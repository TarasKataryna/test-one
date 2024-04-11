# MCC Project


```
public async Task SendTransactionalDataToCfsb(DateTime dateToProcess, Solution solution, bool isLambda)
{
    var methodName = $"[{nameof(SendTransactionalDataToCfsb)}]";
    var currentDateTime = StaticVariables.DateTimeNowEst();
    if (solution != Solution.MoneyExpress && solution != Solution.VisaDirect)
    {
        throw new Exception($"{methodName} Solution should be either MoneyExpress or VisaDirect");
    }
    
    var solutionName = Enum.GetName(typeof(Solution), solution);
    var programId = solution == Solution.MoneyExpress
        ? _mccSettings.CFSBSettings.MeProgramId
        : _mccSettings.CFSBSettings.VdProgramId;

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

        var cfsbCsvProcessor = new CfsbCsvProcessor(_mccSettings);

        var achDetailId = long.Parse(transactions.First().RefNumber);
        var cfsbCsvReportBytes = await cfsbCsvProcessor.ConvertTransactionToTransactionalData(transactions,
            merchants, accounts, currentDateTime, achDetailId, programId);

        await AddLog($"{methodName}[{solutionName}]TRN file was created.");

        var transactionDataLayoutFileName = $"TRN_{_mccSettings.CFSBSettings.CompanyIdentification}_ACH{programId}_{DateTime.UtcNow:yyyyMMddHHmmss}.csv";
        var encryptedDataLayoutFileName = $"{transactionDataLayoutFileName}.pgp";
        try
        {
            var s3ObjectPath = "Settlement/TransactionalDataLayout/Backup/";

            using var backupStream = new MemoryStream(cfsbCsvReportBytes);

            _s3Client = new AWS_S3(_mccSettings, _dbContext);
            await _s3Client.UploadObject($"{s3ObjectPath}{transactionDataLayoutFileName.Replace("/", "-")}", backupStream);

            await AddLog($"{methodName}[{solutionName}]File saved to s3 bucket.");
        }
        catch (Exception ex)
        {
            await AddLog($"{methodName}[{solutionName}][S3Backup] Exception: {ex.Message}", true);
        }

        var sftpFileLocation = $"/users/{_mccSettings.CFSBSettings.SFTPUser}/incoming/{encryptedDataLayoutFileName}";

        _cfsbCommunicationService.CreateSftpClient(isLambda);

        await AddLog($"{methodName}[{solutionName}]SFTP client created.");

        var encryptedFileBytes = _cfsbCommunicationService.EncryptData(isLambda, cfsbCsvReportBytes, encryptedDataLayoutFileName, _mccSettings.CFSBSettings.PGPEncPublic);

        await AddLog($"{methodName}[{solutionName}]Encrypting file.");

        var uploadResult = await _cfsbCommunicationService.UploadFile(sftpFileLocation, encryptedFileBytes, "Customer Data Layout Upload");

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

    var header = new CfsbTransactionalDataHeader()
    {
        RecordType = HeaderRecordType,
        FileType = _mccSettings.CFSBSettings.TransactionalDataLayout.FileType,
        Version = _mccSettings.CFSBSettings.TransactionalDataLayout.Version,
        ProgramType = "ACH",
        CompanyID = _mccSettings.CFSBSettings.MeCompanyIdentification, // todo: clarify source of this value
        ProgramID = programId,
        FileDateTime = dateTimeNow,
    };

    var customerDataCollection = transactions.Select(item =>
    {
        var merchant = merchants.FirstOrDefault(m => m.MerchantID == item.MerchantId);
        var bankAccount = accounts.FirstOrDefault(b => b.MerchantUid == merchant?.Uid);

        var transactionAmount = item.TransactionAmount;

        transactionAmountTotal += transactionAmount;

        return CfsbTransactionalDataDetails.FromMerchant(
            merchant,
            DataRecordType,
            _mccSettings.CFSBSettings.CompanyIdentification,
            programId,
            bankAccount?.AccountNumber,
            item,
            achDetailId
        );
    });

    var trailer = new CfsbTransactionalDataTrailer
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
    csvWriter.WriteRecords(new List<CfsbTransactionalDataHeader> { header });
    csvWriter.WriteRecords(customerDataCollection);
    csvWriter.WriteRecords(new List<CfsbTransactionalDataTrailer> { trailer });

    csvWriter.Flush();

    return memoryStream.ToArray();
}
```
