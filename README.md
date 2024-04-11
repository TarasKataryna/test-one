# MCC Project


```
public async Task SendTransactionalDataToBank(DateTime dateToProcess, Solution solution, bool isLambda)
{
    var methodName = $"[{nameof(SendTransactionalDataToBank)}]";
    var currentDateTime = StaticVariables.DateTimeNowEst();
    if (solution != Solution.FirstSolution && solution != Solution.SecondSolution)
    {
        throw new Exception($"{methodName} Solution should be either FirstSolution or SecondSolution");
    }
    
    var solutionName = Enum.GetName(typeof(Solution), solution);
    var programId = solution == Solution.MoneyExpress
        ? _mccSettings.BankSettings.FirstSolutionProgramId
        : _mccSettings.BankSettings.SecondSolutionProgramId;

    try
    {
        var transactions = await _remittanceTransactionFundingProvider.GetRemittanceTransactionFundingAsync(dateToProcess, dateToProcess);

        await AddLog($"{methodName}[{solutionName}]Transactions to send count - {transactions.Count()}.");

        if (!transactions.Any())
        {
            return;
        }

        var merchants = await _merchantProvider.
            GetAllByConditionAsync($" {nameof(Merchant.MerchantID)} IN ({string.Join(",", transactions.Select(t => "'" + t.MerchantId + "'"))})");

        var accounts = await _merchantAccountProvider.
            GetAllByConditionAsync($" {nameof(MerchantAccount.MerchantUid)} IN ({string.Join(",", merchants.Select(m => m.Uid))})");

        var bankCsvProcessor = new BankCsvProcessor(_mccSettings);

        var achDetailId = long.Parse(transactions.First().RefNumber);
        var bankCsvReportBytes = await bankCsvProcessor.ConvertTransactionToTransactionalData(transactions,
            merchants, accounts, currentDateTime, achDetailId, programId);

        await AddLog($"{methodName}[{solutionName}]TRN file was created.");

        var transactionDataLayoutFileName = $"TRN_{_mccSettings.BankSettings.CompanyIdentification}_ACH{programId}_{DateTime.UtcNow:yyyyMMddHHmmss}.csv";
        var encryptedDataLayoutFileName = $"{transactionDataLayoutFileName}.pgp";
        try
        {
            var s3ObjectPath = "Settlement/TransactionalDataLayout/Backup/";

            using var backupStream = new MemoryStream(bankCsvReportBytes);

            _s3Client = new AWS_S3(_mccSettings, _dbContext);
            await _s3Client.UploadObject($"{s3ObjectPath}{transactionDataLayoutFileName.Replace("/", "-")}", backupStream);

            await AddLog($"{methodName}[{solutionName}]File saved to s3 bucket.");
        }
        catch (Exception ex)
        {
            await AddLog($"{methodName}[{solutionName}][S3Backup] Exception: {ex.Message}", true);
        }

        var sftpFileLocation = $"/users/{_mccSettings.BankSettings.SFTPUser}/incoming/{encryptedDataLayoutFileName}";

        _bankCommunicationService.CreateSftpClient(isLambda);

        await AddLog($"{methodName}[{solutionName}]SFTP client created.");

        var encryptedFileBytes = _bankCommunicationService.EncryptData(isLambda, bankCsvReportBytes, encryptedDataLayoutFileName, _mccSettings.BankSettings.PGPEncPublic);

        await AddLog($"{methodName}[{solutionName}]Encrypting file.");

        var uploadResult = await _bankCommunicationService.UploadFile(sftpFileLocation, encryptedFileBytes, "Customer Data Layout Upload");

        await AddLog($"{methodName}[{solutionName}]File upload result - {uploadResult.Success}");

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


```
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
        FileType = _mccSettings.BankSettings.TransactionalDataLayout.FileType,
        Version = _mccSettings.BankSettings.TransactionalDataLayout.Version,
        ProgramType = "ACH",
        CompanyID = _mccSettings.BankSettings.MeCompanyIdentification, // todo: clarify source of this value
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
            _mccSettings.BankSettings.CompanyIdentification,
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

# RSC Project
```
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

                    var firstServiceOptions = serviceProvider.GetService<IOptions<LexisNexisSettings>>();
                    if (jobConfig is LexisNexisSettings partnerLexisNexisConfigs)
                    {
                        firstServiceObj = new FirstService(Options.Create(partnerLexisNexisConfigs));
                    }
                    else
                    {
                        firstServiceObj = new BridgerXgService(lexisNexisOptions);
                    }

                    var reportGenerator = serviceProvider.GetService<IFirstJobReportGenerator>();
                    var documentRepresentationFactWcs = serviceProvider.GetService<IDocumentRepresentationFactory>();
                    
                    var newFirstJobHandler = new FirstJobJobHandler(wcs, firstServiceObj, reportGenerator, documentRepresentationFactWcs);

                    return newFirstJobHandler;
                    break;
                case EJob.CreditReport:

                    var crs = jobHandler as CreditReportJobHandler;
                    BusinessOwnerService businessOwnerService = null;
                    ExperianHttpClient experianHttpClient = null;

                    var experianOptions = serviceProvider.GetService<IOptions<ExperianSettings>>();

                    if (jobConfig is ExperianSettings partnerExperianConfigs)
                    {
                        var expNewOptions = Options.Create(partnerExperianConfigs);
                        experianHttpClient = new ExperianHttpClient(httpClientFactory, expNewOptions);
                        businessOwnerService = new BusinessOwnerService(experianHttpClient, expNewOptions);
                    }
                    else
                    {
                        experianHttpClient = new ExperianHttpClient(httpClientFactory, experianOptions);
                        businessOwnerService = new BusinessOwnerService(experianHttpClient, experianOptions);
                    }

                    var documentRepresentationFactCr = serviceProvider.GetService<IDocumentRepresentationFactory>();
                    var newCrs = new CreditReportJobHandler(crs, businessOwnerService, documentRepresentationFactCr);

                    return newCrs;
                    break;
                case EJob.ValidiFi:

                    var vs = jobHandler as ValidiFiJobHandler;
                    ValidiFiService validifyService = null;

                    var validifyOptions = serviceProvider.GetService<IOptions<ValidiFiSettings>>();
                    if (jobConfig is ValidiFiSettings partnerValidifyConfigs)
                    {
                        validifyService = new ValidiFiService(Options.Create(partnerValidifyConfigs), httpClientFactory);
                    }
                    else
                    {
                        validifyService = new ValidiFiService(validifyOptions, httpClientFactory);
                    }

                    var newVs = new ValidiFiJobHandler(vs, validifyService);

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

```
public class JobCredentialsManager : IJobCredentialsManager
{
    private const string JobCredsPrefix = "_JobCreds";
    
    private readonly IMemoryCache _memoryCache;
    private readonly IIdentityService _identityService;
    private readonly IMccService _mccService;
    private readonly ILogger<JobCredentialsManager> _logger;

    private readonly ExperianSettings _experianSettings;
    private readonly LexisNexisSettings _lexisNexisSettings;
    private readonly ValidiFiSettings _validiFiSettings;

    public JobCredentialsManager(IMemoryCache memoryCache, 
        IIdentityService identityService, 
        IMccService mccService,
        ILogger<JobCredentialsManager> logger,
        IOptions<ExperianSettings> experianSettings,
        IOptions<LexisNexisSettings> lexisNexisSettings,
        IOptions<ValidiFiSettings> validiFiSettings)
    {
        _memoryCache = memoryCache;
        _identityService = identityService;
        _mccService = mccService;
        _logger = logger;
        _experianSettings = experianSettings.Value;
        _lexisNexisSettings = lexisNexisSettings.Value;
        _validiFiSettings = validiFiSettings.Value;
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
                var partners = await _mccService.GetPartnerHierarchy(partnerId);

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
                            AbsoluteExpiration = DateTimeOffset.UtcNow.AddHours(2)
                        });

                        partnerJobCredentials = configurations.FirstOrDefault(p => p.JobType == (byte)jobType)?.JobConfiguration;
                    }
                }
            }

            switch (jobType)
            {
                case EJob.CreditReport:

                    var expSettings = _experianSettings;

                    if (!string.IsNullOrEmpty(partnerJobCredentials))
                    {
                        var partnerExpSettings = partnerJobCredentials.TryDeserializeObject<ExperianSettings>();
                        
                        if (partnerExpSettings is not null)
                        {
                            expSettings.ClientID = partnerExpSettings.ClientID;
                            expSettings.ClientSecret = partnerExpSettings.ClientSecret;
                            expSettings.Username = partnerExpSettings.Username;
                            expSettings.Password = partnerExpSettings.Password;
                            expSettings.Subcode = partnerExpSettings.Subcode;
                        }
                    }

                    jobCredentials = expSettings;
                    break;
                case EJob.WorldCompliance:

                    var lexisNexisSettings = _lexisNexisSettings;

                    if (!string.IsNullOrEmpty(partnerJobCredentials))
                    {
                        var partnerNexisSettings = partnerJobCredentials.TryDeserializeObject<LexisNexisSettings>();

                        if (partnerNexisSettings is not null)
                        {
                            lexisNexisSettings.ClientId = partnerNexisSettings.ClientId;
                            lexisNexisSettings.UserId = partnerNexisSettings.UserId;
                            lexisNexisSettings.Password = partnerNexisSettings.Password;
                        }
                    }

                    jobCredentials = lexisNexisSettings;
                    break;
                case EJob.ValidiFi:

                    var validifySettings = _validiFiSettings;

                    if (!string.IsNullOrEmpty(partnerJobCredentials))
                    {
                        var partnerValidifySettings = partnerJobCredentials.TryDeserializeObject<ValidiFiSettings>();

                        if (partnerValidifySettings is not null)
                        {
                            validifySettings.ApiKey = partnerValidifySettings.ApiKey;
                            validifySettings.ApiPassword = partnerValidifySettings.ApiPassword;
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
