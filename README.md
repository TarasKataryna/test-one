# Internal records processing and report creation AWS lambda function logic 
**Description**
<br>In this example, the logic demonstrated is generic and can handle different data depending on the parameter passed from the banking program. It retrieves the necessary data, converts it into a CSV file, saves a backup, encrypts this file, and transfers it to the bank's SFTP server, essentially for reporting purposes. Additionally, this logic is specifically for an AWS Lambda function.
<br>In *ConvertTransactionToTransactionalData* method i use *using* keyword to prevent any memory leaks and to clean up unmanaged resources.
```csharp
public async Task SendTransactionalDataToBank(DateTime dateToProcess, Solution solution, bool isLambda)
{
    var methodName = $"[{nameof(SendTransactionalDataToBank)}]";
    var currentDateTime = StaticVariables.DateTimeNowEst();
    if (solution != Solution.FirstSolution && solution != Solution.SecondSolution)
    {
        throw new Exception($"{methodName} Solution should be either FirstSolution or SecondSolution");
    }
    
    var solutionName = Enum.GetName(typeof(Solution), solution);
    var programId = solution == Solution.FirstSolution
        ? _internalSettings.BankSettings.FirstSolutionProgramId
        : _internalSettings.BankSettings.SecondSolutionProgramId;

    try
    {
        var transactions = await _remittanceTransactionFundingProvider.GetRemittanceTransactionFundingAsync(dateToProcess, dateToProcess);

        if (!transactions.Any())
        {
            return;
        }

        var merchants = await _merchantProvider.
            GetAllByConditionAsync($" {nameof(Merchant.MerchantID)} IN ({string.Join(",", transactions.Select(t => "'" + t.MerchantId + "'"))})");

        var accounts = await _merchantAccountProvider.
            GetAllByConditionAsync($" {nameof(MerchantAccount.MerchantUid)} IN ({string.Join(",", merchants.Select(m => m.Uid))})");

        var bankCsvProcessor = new BankCsvProcessor(_internalSettings);

        var achDetailId = long.Parse(transactions.First().RefNumber);
        var bankCsvReportBytes = await bankCsvProcessor.ConvertTransactionToTransactionalData(transactions,
            merchants, accounts, currentDateTime, achDetailId, programId);

        var transactionDataLayoutFileName = $"TRN_{_internalSettings.BankSettings.CompanyIdentification}_ACH{programId}_{DateTime.UtcNow:yyyyMMddHHmmss}.csv";
        var encryptedDataLayoutFileName = $"{transactionDataLayoutFileName}.pgp";
        try
        {
            var s3ObjectPath = "Settlement/TransactionalDataLayout/Backup/";

            using var backupStream = new MemoryStream(bankCsvReportBytes);

            _s3Client = new AWS_S3(_internalSettings, _dbContext);
            await _s3Client.UploadObject($"{s3ObjectPath}{transactionDataLayoutFileName.Replace("/", "-")}", backupStream);
        }
        catch (Exception ex)
        {
            await AddLog($"{methodName}[{solutionName}][S3Backup] Exception: {ex.Message}", true);
        }

        var sftpFileLocation = $"/users/{_internalSettings.BankSettings.SFTPUser}/incoming/{encryptedDataLayoutFileName}";

        _bankCommunicationService.CreateSftpClient(isLambda);

        var encryptedFileBytes = _bankCommunicationService.EncryptData(isLambda, bankCsvReportBytes, encryptedDataLayoutFileName, _internalSettings.BankSettings.PGPEncPublic);

        var uploadResult = await _bankCommunicationService.UploadFile(sftpFileLocation, encryptedFileBytes, "Customer Data Layout Upload");


        if (!uploadResult.Success)
        {
            throw new Exception(uploadResult.Message);
        }
    }
    catch (Exception ex)
    {
        await AddLog($"{methodName}[{solutionName}] Exception: {ex.Message}", true);
    }
}
```


```csharp
public async Task<byte[]> ConvertTransactionToTransactionalData(
    IEnumerable<RemittanceTransactionFunding> transactions, 
    IEnumerable<Merchant> merchants, 
    IEnumerable<MerchantAccount> accounts,
    DateTime postDate,
    long achDetailId,
    string programId)
{
    decimal transactionAmountTotal = 0;
    var dateTimeNow = DateTime.UtcNow.ToString("yyyy-MM-dd hh:mm:ss tt");

    var header = new BankTransactionalDataHeader()
    {
        RecordType = HeaderRecordType,
        FileType = _internalSettings.BankSettings.TransactionalDataLayout.FileType,
        Version = _internalSettings.BankSettings.TransactionalDataLayout.Version,
        ProgramType = "ACH",
        CompanyID = _internalSettings.BankSettings.MeCompanyIdentification, // todo: clarify source of this value
        ProgramID = programId,
        FileDateTime = dateTimeNow,
    };

    var customerDataCollection = transactions.Select(item =>
    {
        var merchant = merchants.FirstOrDefault(m => m.MerchantID == item.MerchantId);
        var bankAccount = accounts.FirstOrDefault(b => b.MerchantUid == merchant?.Uid);

        var transactionAmount = item.TransactionAmount;

        transactionAmountTotal += transactionAmount;

        return BankTransactionalDataDetails.FromMerchant(
            merchant,
            DataRecordType,
            _internalSettings.BankSettings.CompanyIdentification,
            programId,
            bankAccount?.AccountNumber,
            item,
            achDetailId
        );
    });

    var trailer = new BankTransactionalDataTrailer
    {
        RecordType = TrailerRecordType,
        RecordCount = customerDataCollection.Count(),
        TransactionAmountTotal = $"{transactionAmountTotal:0.00}"
    };

    using var memoryStream = new MemoryStream();
    using var streamWriter = new StreamWriter(memoryStream);
    using var csvWriter = new CsvWriter(streamWriter, new CsvConfiguration(CultureInfo.InvariantCulture)
    {
        Delimiter = Delimiter,
        HasHeaderRecord = false
    });

    //in order to have new line between records - put to list (or call NextRecord() method)
    csvWriter.WriteRecords(new List<BankTransactionalDataHeader> { header });
    csvWriter.WriteRecords(customerDataCollection);
    csvWriter.WriteRecords(new List<BankTransactionalDataTrailer> { trailer });

    csvWriter.Flush();

    return memoryStream.ToArray();
}
```


# Custom job handler resolving with creds caching
# Custom Json Converter
**Description**
<br> The main purpose of this implementation was to extend the logic of resolving the necessary implementations of background jobs, which mainly contain logic for third-party services integrations. To the existing logic, which included dependency injection of these background jobs and their resolving with ready-made services and configurations, it was necessary to add the ability to resolve these jobs with configurations that are contained in another project depending on the partner that invokes this logic. In this code snippet, two classes are provided, *JobHandlerFactory* for resolving these jobs with a dependency injection container and modifying the services and configurations they need to continue working. This class calls a conditional creds manager - *JobCredentialsManager*, which makes requests to another service, retrieves configurations, caches them if necessary, and returns them. As a result, we can use jobs with configurations specified in the appsettings.json file, as well as configurations that need to be obtained during program execution
```csharp
public class JobHandlerFactory: IJobHandlerFactory
{
    private readonly IJobCredentialsManager _jobCredentialsManager;

    public JobHandlerFactory(IJobCredentialsManager jobCredentialsManager)
    {
        _jobCredentialsManager = jobCredentialsManager;
    }

    public async Task<IJobHandler> GetJobHandler(EJob jobType, IServiceScope scope, string partnerId)
    {
        var jobHandler = scope.ServiceProvider.GetServices<IJobHandler>()
            .First(x => x.JobType == jobType);

        jobHandler = await ResolveJobHandler(jobType, partnerId, jobHandler, scope.ServiceProvider);

        return jobHandler;
    }

    public async Task<IJobHandler> GetJobHandler(EJob jobType, IHttpContextAccessor accessor, string partnerId)
    {
        var jobHandler = accessor.HttpContext?.RequestServices.GetServices<IJobHandler>()
            .First(x => x.JobType == jobType);

        if (!string.IsNullOrEmpty(partnerId?.Trim()))
        {
            jobHandler = await ResolveJobHandler(jobType, partnerId, jobHandler, accessor.HttpContext?.RequestServices);
        }

        return jobHandler;
    }

    private async Task<IJobHandler> ResolveJobHandler(EJob jobType, string partnerId, IJobHandler jobHandler, IServiceProvider serviceProvider)
    {
        if (jobType is EJob.FirstJob or EJob.SecondJob or EJob.ThirdJob)
        {
            var jobConfig = await _jobCredentialsManager.Get(partnerId, jobType);
            var httpClientFactory = serviceProvider.GetService<IHttpClientFactory>();

            switch (jobType)
            {
                case EJob.FirstJob:

                    var oldFirstJobHandler  = jobHandler as FirstJobJobHandler;
                    FirstService firstServiceOptions = null;

                    var firstServiceOptions = serviceProvider.GetService<IOptions<FirstServiceSettings>>();
                    if (jobConfig is FirstServiceSettings partnerFirstServiceConfigs)
                    {
                        firstServiceObj = new FirstService(Options.Create(partnerFirstServiceConfigs));
                    }
                    else
                    {
                        firstServiceObj = new BridgerXgService(firstServiceOptions);
                    }

                    var reportGenerator = serviceProvider.GetService<IFirstJobReportGenerator>();
                    var documentRepresentationFactWcs = serviceProvider.GetService<IDocumentRepresentationFactory>();
                    
                    var newFirstJobHandler = new FirstJobJobHandler(wcs, firstServiceObj, reportGenerator, documentRepresentationFactWcs);

                    return newFirstJobHandler;
                    break;
                case EJob.SecondJob:

                    var secondJobHandler = jobHandler as SecondJobHandler;
                    SecondService secondService = null;
                    SecondServiceHttpClient secondServiceHttpClient = null;

                    var secondServiceOptions = serviceProvider.GetService<IOptions<SecondServiceSettings>>();

                    if (jobConfig is SecondServiceSettings partnerSecondServiceConfigs)
                    {
                        var expNewOptions = Options.Create(partnerSecondServiceConfigs);
                        secondServiceHttpClient = new SecondServiceHttpClient(httpClientFactory, partnerSecondServiceConfigs);
                        secondService = new SecondService(secondServiceHttpClient, partnerSecondServiceConfigs);
                    }
                    else
                    {
                        secondServiceHttpClient = new SecondServiceHttpClient(httpClientFactory, secondServiceOptions);
                        SecondService = new SecondService(secondServiceHttpClient, secondServiceOptions);
                    }

                    var documentRepresentationFactCr = serviceProvider.GetService<IDocumentRepresentationFactory>();
                    var secondNewJobHandler = new CreditReportJobHandler(secondOldJobHandler, businessOwnerService, documentRepresentationFactCr);

                    return secondNewJobHandler;
                    break;
                case EJob.TirdJob:

                    var vs = jobHandler as ThirdJobHandler;
                    ThirdService thirdService = null;

                    var thirdServiceOptions = serviceProvider.GetService<IOptions<ThirdServiceSettings>>();
                    if (jobConfig is ThirdServiceSettings partnerThirdServiceConfigs)
                    {
                        thirdService = new ThirdService(Options.Create(partnerThirdServiceConfigs), httpClientFactory);
                    }
                    else
                    {
                        thirdService = new ThirdService(thirdServiceOptions, httpClientFactory);
                    }

                    var newVs = new ThirdJobHandler(vs, thirdService);

                    return newVs;
                    break;
                default:
                    break;

            }
        }

        return jobHandler;
    }
}
```

```csharp
public class JobCredentialsManager : IJobCredentialsManager
{
    private const string JobCredsPrefix = "_JobCreds";
    
    private readonly IMemoryCache _memoryCache;
    private readonly IIdentityService _identityService;
    private readonly IInternalService _internalService;
    private readonly ILogger<JobCredentialsManager> _logger;

    private readonly FirstServiceSettings _firstServiceSettings;
    private readonly SecondServiceSettings _secondServiceSettings;
    private readonly ThirdServiceSettings _thirdServiceSettings;

    public JobCredentialsManager(IMemoryCache memoryCache, 
        IIdentityService identityService, 
        IInternalService internalService,
        ILogger<JobCredentialsManager> logger,
        IOptions<FirstServiceSettings> firstServiceSettings,
        IOptions<SecondServiceSettings> secondServiceSettings,
        IOptions<ThirdServiceSettings> thirdServiceSettings)
    {
        _memoryCache = memoryCache;
        _identityService = identityService;
        _internalService = internalService;
        _logger = logger;
        _firstServiceSettings = firstServiceSettings.Value;
        _secondServiceSettings = secondServiceSettings.Value;
        _thirdServiceSettings = thirdServiceSettings.Value;
    }

    public async Task<object> Get(string partnerId, EJob jobType)
    {
        object jobCredentials = null;
        try
        {
            string partnerJobCredentials = null;

            var result = _memoryCache.TryGetValue($"{partnerId}{JobCredsPrefix}", out var partnerBasedCreds);

            if (result)
            {
                var partnerBasedDict = (Dictionary<string, string>)partnerBasedCreds;

                if (partnerBasedDict.TryGetValue(((byte)jobType).ToString(), out var jobCreds))
                {
                    partnerJobCredentials = jobCreds;
                }
            }
            else
            {
                var partners = await _internalService.GetPartnerHierarchy(partnerId);

                if (partners is not null && partners.Any())
                {
                    var request = new PartnerConfigurationRequest
                    {
                        Partners = partners.Select(p => new PartnerModel
                            { PartnerId = p.PartnerId, PartnerType = (byte)p.PartnerType })?.ToList()
                    };

                    var configurations = await _identityService.GetJobsConfigurations(request);

                    if (configurations is not null && configurations.Any())
                    {
                        var credentialsDict = configurations.Where(p => p.JobConfiguration is not null)
                            .ToDictionary(p => p.JobType.ToString(), p => p.JobConfiguration);

                        _memoryCache.Set($"{partnerId}{JobCredsPrefix}", credentialsDict, new MemoryCacheEntryOptions
                        {
                            AbsoluteFirstServiceiration = DateTimeOffset.UtcNow.AddHours(2)
                        });

                        partnerJobCredentials = configurations.FirstOrDefault(p => p.JobType == (byte)jobType)?.JobConfiguration;
                    }
                }
            }

            switch (jobType)
            {
                case EJob.FirstJob:

                    var expSettings = _firstServiceSettings;

                    if (!string.IsNullOrEmpty(partnerJobCredentials))
                    {
                        var partnerFirstServiceSettings = partnerJobCredentials.TryDeserializeObject<FirstServiceSettings>();
                        
                        if (partnerFirstServiceSettings is not null)
                        {
                            expSettings.ClientID = partnerFirstServiceSettings.ClientID;
                            expSettings.ClientSecret = partnerFirstServiceSettings.ClientSecret;
                            expSettings.Username = partnerFirstServiceSettings.Username;
                            expSettings.Password = partnerFirstServiceSettings.Password;
                            expSettings.Subcode = partnerFirstServiceSettings.Subcode;
                        }
                    }

                    jobCredentials = expSettings;
                    break;
                case EJob.SecondJob:

                    var secondServiceSettings = _secondServiceSettings;

                    if (!string.IsNullOrEmpty(partnerJobCredentials))
                    {
                        var partnerSecondServiceSettings = partnerJobCredentials.TryDeserializeObject<SecondServiceSettings>();

                        if (partnerSecondServiceSettings is not null)
                        {
                            secondServiceSettings.ClientId = partnerSecondServiceSettings.ClientId;
                            secondServiceSettings.UserId = partnerSecondServiceSettings.UserId;
                            secondServiceSettings.Password = partnerSecondServiceSettings.Password;
                        }
                    }

                    jobCredentials = secondServiceSettings;
                    break;
                case EJob.ThirdJob:

                    var validifySettings = _thirdServiceSettings;

                    if (!string.IsNullOrEmpty(partnerJobCredentials))
                    {
                        var partnerThirdServiceSettings = partnerJobCredentials.TryDeserializeObject<ThirdServiceSettings>();

                        if (partnerThirdServiceSettings is not null)
                        {
                            validifySettings.ApiKey = partnerThirdServiceSettings.ApiKey;
                            validifySettings.ApiPassword = partnerThirdServiceSettings.ApiPassword;
                        }
                    }

                    jobCredentials = validifySettings;

                    break;
            }

        }
        catch(Exception ex)
        {
            _logger.LogError(ex, $"Error during retrieving job configs for partner - {partnerId}, job - {jobType}");
        }

        return jobCredentials;
    }
}

```


# Custom Json Converter
**Description**
<br> *IndividualInfoIntegrationDto* class has complex structure and is needed to be converted from json differently. *BusinessOwnerConverter* inherits from JsonConverter class and implements it's own logic for *IndividualInfoIntegrationDto* objects converting. 

```csharp
    public class BusinessOwnerConverter : JsonConverter
    {
        private const int IndividualCollectionsMaxCount = 4;

        public override void WriteJson(JsonWriter writer, object value, JsonSerializer serializer)
        {
            throw new NotImplementedException();
        }

        public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)
        {
            var json = JObject.Load(reader);

            foreach (var property in json.Properties().ToList())
            {
                property.Replace(new JProperty(property.Name.ToLower(), property.Value));
            }

            var boDtoType = typeof(BusinessOwnerIntegrationDto);
            var boObjectProperties = boDtoType.GetProperties();

            var individualInfoDto = new IndividualInfoIntegrationDto();
            for (var i = 1; i <= IndividualCollectionsMaxCount; ++i)
            {
                var businessOwner = new BusinessOwnerIntegrationDto();
                foreach (var property in boObjectProperties)
                {
                    var jsonPropertyName = string.Format(property.GetCustomAttribute<JsonPropertyAttribute>().PropertyName, i).ToLower();
                    var objectPropertyType = Nullable.GetUnderlyingType(property.PropertyType) ?? property.PropertyType;

                    var strValue = json[jsonPropertyName]?.ToString()?.Trim();

                    var safeValue = string.IsNullOrEmpty(strValue) ? null : Convert.ChangeType(strValue, objectPropertyType);

                    boDtoType.GetProperty(property.Name)?.SetValue(businessOwner, safeValue);
                }

                individualInfoDto.BusinessOwnerCollection.Add(businessOwner);
            }

            return individualInfoDto;
        }

        public override bool CanConvert(Type objectType)
        {
            return objectType == typeof(IndividualInfoIntegrationDto);
        }
    }
```


# User's permissions validation 
**Description**
<br>All user's data with permissions are stored in the Single Sign-On web applicaiton. This example contains the functionallity that validate user's permission in internall web application calling the SSO application.
*PermissionRequiredFilter* filter contains requests logic to SSO and pass there required permissions taken from the *PermissionAuthorizeAttribute* attribute. 

```csharp
 public class PermissionRequiredFilter : IAsyncAuthorizationFilter
 {
     private const string AuthorizationHeader = "Authorization";
     private const string BearerPrefix = "Bearer ";
     private const string TokenExpired = "Token Expired";

     private readonly IIdentityService _identityService;

     public PermissionRequiredFilter(IIdentityService identityService)
     {
         _identityService = identityService;
     }

     public async Task OnAuthorizationAsync(AuthorizationFilterContext context)
     {
         if (context.ActionDescriptor is ControllerActionDescriptor controllerActionDescriptor)
         {
             if (controllerActionDescriptor.MethodInfo.GetCustomAttribute(typeof(PermissionAuthorizeAttribute)) is PermissionAuthorizeAttribute methodPermissionAttributeData)
             {
                 if (string.IsNullOrEmpty(methodPermissionAttributeData.Permission))
                 {
                     context.Result = new ForbidResult();
                 }
                 else
                 {
                     await PermissionCheckingProcess(context, methodPermissionAttributeData.Permission);
                 }
             }
         }
     }

     private async Task PermissionCheckingProcess(AuthorizationFilterContext context, string permission)
     {
         //Here we expect to receive 200 status code (user has the permission) in order not to block incoming request and process this request
         var permissionCheckStatus = await _identityService.CheckUserPermission(permission);

         switch (permissionCheckStatus)
         {
             case HttpStatusCode.Unauthorized:
                 context.Result = new UnauthorizedObjectResult(TokenExpired);
                 break;
             case HttpStatusCode.NotFound:
                 context.Result = new ForbidResult();
                 break;
         }
     }
 }

[AttributeUsage(AttributeTargets.Method)]
public class PermissionAuthorizeAttribute : AuthorizeAttribute, IFilterFactory
{
    public string Permission { get; set; }

    public bool IsReusable => true;

    public PermissionAuthorizeAttribute(string permission)
    {
        Permission = permission;
    }

    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        return (IFilterMetadata) serviceProvider.GetService(typeof(PermissionRequiredFilter));
    }
}
```
