# test-one
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
