
## What's the design for changing the output formats?


> this is done in `user-asset-report-generator`...
> Because the output format is not structured data like json, or xml, it is plain-text report format, 
> I decided to use a templating framework. I've worked with several java templating frameworks such as freemarker, Velocity, Thymeleaf, and Mustache. 
> My avorite is Mustache, and it has support in most other languages. I would write the lambda in nodejs. 
> The input is xml, then gets translated to json and put on queue. when we consume the message from queue it is in json format so 
> we can just take our structure and apply it to the template. 

## user-asset-translater pattern classes and psuedo code:

In this example both the input model and the output model have the same names but a different namespace, so to make it clearer we can rename the input user based on the provider. In this case I'll just rename the input classes to be ExternalUser

```
package input

class ExternalUser {
    String userId
    ExternalAssetAccount[] assets
}

class ExternalAssetAccount {
    String type
    String accountNumber
    Double amount
    AssetDescription description
}

class AssetDescription {
    String accountNumber
    String bankAccountType
    String retirementAccountType
    String bankName
    int percentStocks
    int percentBonds
    int percentOther
}
```

```
package output
class User {
	String userId
	String externalUserId
	AssetAccount[] assetAccounts
}

class AssetAccount {
	String accountAccountId
	String accountNumber
	Double amount
	AccountTypeEnum accountType
	TaxCodeEnum taxCode
	AssetAccountAllocations[] allocations
}

class AssetAllocation {
	String assetAccountId
	AssetAllocationTypeEnum type
	int percent
}
```

```
interface IExternalUserXMLParser {
    JSON parse(String xmlString)
}
```

```
class ExternalUserXMLParser {
  
  JsonNode parse(String xml){
    JsonNode json = parser.toJson(xml);
  }
}
```

```
class Json2ExternalUserMapper {
    ExternalUser map(JsonNode jsonNode){

        User usse= new ArrayList<User>;
        List<ExternalAssetAccounts> assetAccounts = mapExternalAssetAcounts(jsonNode.get('assets));
        user.setAssetAccounts(assetAccounts)
        
        // etc

        return users
    }

    External User mapExternalAssetAccounts(JSON json){
        // extract the externalAssetAccounts
    }
}
```

```
class UserAssetEventTranslator {

    IExternalUserXMLParser xmlParser = new ExternalUserXMLParser();

    Json2UserMapper json2UserMapper = new Json2UserMapper()

    ExternalUserValidator validator = new ExternalUserValidator()

    List<User> translate(String xmlPayload){
        JsonNode json = xmlParser.parse(xmlPayload)
        List<Users> users = json2UserMapper.map(json);
        users.forEach(user -> 
            validator.validate(user)
        )
     }

}
```

```
class ExternaUserInputValidator {

    validate (ExternaUser user){
        // validate user fields
        user.getAccounts().forEach(account -> 
          validateAccount(account)
        )
    }

    validateAccount(ExternaAssetAccount account){
      // validate account fields
    }
}
```

```
class InputUser2OutputUserMapper {

    output.User map (input.User inputUser){
       output.User user = new output.User()
       user.setExternalUserId(inputUser.getUserId())
       user.setUserId(uuid())
       user.setAssetAccounts(mapAssetAccounts(inputUser))
    }

    List<output.AssetAccount> mapAssetAccounts(input.User()){
       //for each map to the output format. 
    }
}
```

## asset-report-generator pattern classes and psuedo code:

```
class UserAssetReport {
    List<AssetAccount> accounts;
    List<IReportAggregation> aggregations = Arrays.asList(
                                new ReportSumAggregation()
                                new ReportMeanAggregation()
    )
    
    add(AssetAccount assetAccount){
        accounts.add(assetAccount)
        aggregations.forEach(aggregation -> {
            aggregation.add(assetAccount)
        })
    }
}
```

```
interface IReportAggregation<T> {
	String getType()
	add(AssetAccount userAsset)
	T calcuclate()
}
```

```
class ReportSumAggregation implements IReportAggregation<Double> {

	List<Double> values = new ArrayList<>();

	String getType() 
	{
	  return "Total"
	}
	
	add(AssetAccount assetAcount){
		values.add(assetAccount.getAmount())
	}

	Double calcuclate(){
		Double sum = 0
		values.forEach(val){
			sum = sum + val
		}
		return sum
	}
}
```

```
class ReportMeanAggregation implements IReportAggregation<Double> {

	List<Doubles> values = new ArrayList<>();

	String getType() 
	{
	  return "Total"
	}
	
	add(AssetAccount assetAcount){
		values.add(assetAccount.getAmount())
	}

	Double calcuclate(){
		Double sum = 0
		values.forEach(val){
			sum = sum + val
		}
		return sum / values.count()
	}
}
```

```
interface ITemplateEngine {
	apply(pathToTemplate,  List<AssetAccount> assetAccounts)
}
```

```
class MustacheTemplateEngine {
	String apply(pathToTemplate,  AssetReport report){
		fileContent = mustacheAPI.apply(pathToTemplate, report)
		return fileContent
	}
}
```

```
class SummaryReportGenerator {
	
	Map<String, String> templates = new HashMap();

	static 
	{
		templates.put('primary', 'path/to/asset-report.mustache' ) 
	}
	
	ITemplateEngine templateEngine = new MustachTemplateEngine()

	UserAssetReport report = new UserAssetReport()

	Stirng generate(List<User> users, String templateName){

		users.forEach((user) -> {
			user.getAssetAccounts().forEach((assetAccount)-> {
				report.getAssetAccounts().add(assetAccount)
				report.aggregate(assetAccount)
			})
		})

		String fileContent = templateEngine.apply(templates.get(templateName), report)

		return fileContent;
	}
}
```

```
class LambdaHandler {

    S3Client s3 = new S3Client()

	handler(event){
		SummaryReportGenerator reportGen = new SummaryReportGenerator()
		String fileContent = reportGen.generate(event.users, event.templateName)
		
		s3.putObject(fileContent)
	}
}
```


What's the design for adding new filtered aggregates?

To add a new filtered aggregate, you just need to 
   (1) create another implementation of IReportAggregation
   (2) add the concrete class to the UserAssetReport's list of IReportAggregation

  it will automatically get added to the summary section

What's the design for adding new types of assets? 
	For a new Account Type support you would need to add to the enum list, and the MoneyMarketAccountTaxCode or null if not applicable
	A new asset would also conform to the AssetAccount class. For example, lets use Money market account

   AssetAccount moneyMarketAccount = new AssetAccount();

   moneyMarketAccount.setAccountAccountId(uuid())
   moneyMarketAccount.setAccountNumber()
   moneyMarketAccount.setAmount(100000D)
   moneyMarketAccount.setAccountType(AccountTypes.MONEY_MARKET)
   moneyMarketAccount.setTaxCode(null)


What's the design for serializing output?

this is done in `user-asset-report-generator`...
Because the output format is not structured data like json, or xml, it is plain-text report format, 
I decided to use a templating framework. I've worked with several java templating frameworks such as freemarker, Velocity, Thymeleaf, and Mustache. My favorite is Mustache, and it has support in most other languages. I would write the lambda in nodejs. The input is xml, then gets translated to json and put on queue. when we consume the message from queue it is in json format so we can just take our structure and apply it to the template. 

What's the organization, or key naming convention, in the user-asset-inbox/processed and user-asset-inbox/errors buckets?

The file name for /errors and /processed would not diff from the original file added.
the file format should have a name and ISO 8601 timestamp. 
 Example Input: 

   USER_ASSET_FILE_2020-08-04T05:22:52+0000.xml

 Example Input: 

   USER_ASSET_REPORT_2020-08-04T05:23:52+0000.txt


How does user-asset-event-processor split batches into single entries?
 
   The file content is first deserialized into a collection, them split in code. 

Is the report generator keeping the report in memory until the whole file is ready to be written to disk and available for pick-up?  

   Yes, if we load test it and have any problems we could split the reports based on max file size

The signal for report to be generated is a two minute interval between requests.  If we receive a request every minute for an hour and then nothing for two minutes, then the report will have data from 60 requests.  How is this implemented?

   A dry walktrough of this scenario would be like this:

   0m
   +1m file n(1) received
   +2m file n(2) received
   [first report generated]
   +3m file n(3)received
   +4m file n(4) received
   [second report generated]

   Or do you mean we would receive a request every second?

   0m
   +1s file n(1) received
   +2s file n(2) received
   +3s file n(3) received
   ...
   +60s file n(60) received
   [first report generated]
   +61s file n(61) received
   +62s file n(62) received
   ...
   +120s file n(120) received 
   [second report generated]

   In the case where we have 60 asset events on queue, then we would process all 60 unless we decided to have a max records consumed set. The report-generator consumes every 2 min all messages on queue, so however many are there it should be able to process them.



Where does the data for retirementAccountType go?  Is the AssetAccount class used to hold data from both brokerageAccount and bankAccount elements?


Performance

Does it make sense to scale any part of this?  If the answer is yes, that may affect output ordering, in which case you should describe how ordering is preserved. 

Out put ordering is preserved because of the queue is set to have FIFO (first in first out ordering). If we scale the input, it will essentially throttle based on the maxRecordssetting of the report generator. The lambda would originally be configured to only allow 1 active instance. We could scale the report generation by allowing concurrent invokations of the lambda. in this case, we can ensure that files are put in order by adding an temporary holding folder for completed reports to go in, then another queue, that has a message added to it to poll for a specific file to exist before moving to the outbox. Alternativly, we can add a sequence number within the report and instruct the consumer to not process out-of-order. 


Validation

Please point out exactly where in those processes validations will occur.  For example they could occur before parsing, after parsing, during parsing, or other places.  You also have a list of validations.  Point out if all of these will be done in the same part of the process.  If not, which validations will go where. 


user-asset-event-processor
--------------------------
1. catch serializaion errors and throw error
2. validate when payload or file is received after splitting
3. then publish to queue

asset-report-generator
--------------------------
1. catch serializaion errors and throw error
2. validate message when message is consumed
3. then generate report

* note: always pre-validate before processsing

You mentioned validation in user-asset-event-processor and user-asset-report-generator.  Are those different validations? 

You mentioned validation against both the original form and the normalized form.  I would think you'd validate original, then convert original to normalized.  Is that correct or would you validate normalized as well?

This depends on if we can forsee any reason to access the user-asset-report-generator from another enty point or from a replay mechanism.  If so, I think it's important to validate in both because we may have mutliple entry points into the same process. The validations done against the user input so that the error messages are in the context of the input, rather than the translated internal representation. 


Status Codes

You listed various ways to get a 400 for validation. but there are all kinds of other possible status codes.  List any non-validation ones and describe how their error message (for those that are errors) will be populated.  For example you could get a 500 for various reasons.  

Pre parsing errors:
  - 401 Unauthorized: client error status response code indicates that the request has not been applied because it lacks valid authentication credentials for the target resource.
  - 403 Forbidden:  client error when they do not have access rights to the resource
  - 405 Method Not Allowed: for example, if they try to do a GET instead of POST,
  - 413 Payload Too Large 
Parsing errors:  
  - 500 Service Error - any time the process unexpectantly fails
  - 504 Gateway Timeout - when the processing takes too long
After Parsing
  - 500 Service Error - if publishing to queue fails or any other coding error NullPointer, etc


What's the strategy in your code for a unified way to serialize different kinds of exceptions into structured data in the response?

API exceptions
--------------

  should match format: 
  {
    statusCode: 400,
    errorCode: "MANDATORY_FIELD"
    message: "Missing Asset Account Amount for User '121243645' Account Number '6546453645'",
  }

Are there limitations in your system for these status codes?  For example, are there scenarios where a request can't be processed, but a 200 is returned?  It's perfectly fine to have limitations, but you should explain why the benefit is greater than the cost.  

 - When an xml message is received it is processed in realtime, and the response code will only return 200 if no error has occured an the message has been put on queue.
 - When an xml file is received, the sftp will give a success response when the file is uploaded, even if no processing is done yet. 





