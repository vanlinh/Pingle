 public static async Task<Result> ProcessPaymentAsync(PaymentRequest requestPayment, DataContext content)
        {
            var config = new BraintreeConfiguration();
            var gateway = config.GetGateway();
            var request = new TransactionRequest
            {
                Amount = requestPayment.Amount,
                Options = new TransactionOptionsRequest
                {
                    SubmitForSettlement = requestPayment.SubmitForSettlement
                }
            };

            //Save payment method 
            if (requestPayment.SavePaymentMethod)
            {
                Customer customer;
                string token = "";
                if (string.IsNullOrEmpty(requestPayment.CustomerId))
                {
                    UserInfoWithDetail userinfo = await content.UserInfoWithDetails.FindAsync(requestPayment.UserId);
                    var cusrequest = new CustomerRequest
                    {
                        FirstName = userinfo.FirstName,
                        LastName = userinfo.LastName,
                        PaymentMethodNonce = requestPayment.Nonce

                    };
                    Result<Customer> cusresult = await gateway.Customer.CreateAsync(cusrequest);
                    bool success = cusresult.IsSuccess();
                    if (success)
                    {    
                        var newinfo = new UserwithPaymentMethod();
                        newinfo.CustomerId = cusresult.Target.Id;
                        newinfo.UserId = requestPayment.UserId;
                        newinfo.Gateway = "BRAINTREE";
                        newinfo.CreatedTimeStamp = DateTime.UtcNow;
                        content.UserwithPaymentMethods.Add(newinfo);
                        await content.SaveChangesAsync();
                    }
                    customer = cusresult.Target;
                    token = customer.PaymentMethods[0].Token;
                }
                else
                {
                    customer = await gateway.Customer.FindAsync(requestPayment.CustomerId);
                    var requestnewMethod = new PaymentMethodRequest
                    {
                        CustomerId = customer.Id,
                        PaymentMethodNonce = requestPayment.Nonce
                    };

                    Result<PaymentMethod> newmethodresult = gateway.PaymentMethod.Create(requestnewMethod);
                    token = newmethodresult.Target.Token;

                }
                //Make payment with token
                request.PaymentMethodToken = token;
            }
            //Not save Payment Method
            else
            {
                request.PaymentMethodNonce = requestPayment.Nonce;
                request.Options.StoreInVaultOnSuccess = false;
            }

            var rs = new Result();

            if (!requestPayment.IsPaypal)
            {
                request.MerchantAccountId = requestPayment.MerchantId;
                request.ServiceFeeAmount = requestPayment.ServiceFeeAmount;
            }
            else
            {
                //Todo: Have to save this transaction is paypal payments
                //
            }

            Result<Transaction> result = await gateway.Transaction.SaleAsync(request);
            if (result.IsSuccess())
            {
                //Todo: await PaymentSucceed(requestPayment.OrderId)
                //Insert to a queue to release money from escrow if it is not a paypal payment

                Transaction transaction = result.Target;
                rs.Success = true;
                rs.Value = result.Target.Id; //transactionid  

            }
            else if (result.Transaction != null)
            {
                rs.Success = false;
                rs.ErrorMessage = result.Errors.ToString(); //Todo: Need to handle
            }
            else
            {
                //Todo: await PaymentFailed(OrderId)
                string errorMessages = "";
                foreach (ValidationError error in result.Errors.DeepAll())
                {
                    errorMessages += "Error: " + (int)error.Code + " - " + error.Message + "\n";
                }
                rs.Success = false;
                rs.ErrorMessage = errorMessages; //Todo: Need to handle

            }
            return rs;

        }
