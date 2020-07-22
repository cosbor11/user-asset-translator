hitarth


Questions:
----------

1. Why can a User only have one RetirementAccount and one BrokerageAccount even though this is not a real life constraint?
    - can have multiple

2. Why are the BankAccounts not grouped together in the Report Output? What is the expected sort order.
   - expected output must top down order (match the input)

3. What is the transmission protocol for receiving (xml), and submitting (plaintext), should I assume http(s)?
   Input: 
   	- two ways: 
   		1) https over rest server (single user record),
   		2) batch over sftp (endpoint for inbox, endpoint for outbox)
   		  - file appears batch 
   		  - sequence matters 
   		  - files can be processed at 2am (every 10sec)



5. Is the xml being being uploaded in an multipart file or transmitted as an exchange body?
  body xml
6. What are the delivery requirements (realtime, within an hour/day)?
    http must respond with reasonable within 1 min

7. What is the peek load and expected capacity?
    100 sec
    6k request per min


8. how to handle requests return error message, no need to implement rate limiting

9. Is the Reporting System an external party or within the same VPN?
	put self served ftp server in outbox folder.
	- file should stay 
	- file naming convension (uniqueness)
	- describe file name contract in solution

9. Should we store the information provided in XML in a database (if so, how long will it need to be stored?)
 - no persistence, must log errors
10. What output should be displayed when a user does not have any accounts, or is missing one of them?
   - {} - must have at least one asset
     - log, error with code & message (list error cases with codes, and messsages, scenarios)
     - sftp, just log

11. Is a description mandatory for all account types? (if so then assetDescriptionId I prefer it to be on AssetAccount, rather than on the )
     - description is maditory
     all is manditory - must have at least one, all asset accounts can be multiple 

12. Are all currencies in USD (840)?
	yes, rouded to near dollar (no cents)
	reject format 45.95 invalid

13. Is userId an identifier in the external system or in our system?
     cannot verify validity of user (must have length 10-30 digits, could use REGEX)

14. What precision should the stock percentage hold?
      0-100
15. What max precision for account amount should we hold?
      0-1 trillion
16. What is the required uptime availabilty?
     highly available, 
17. What are the cost or time contraints?
    none, but must have good architecture (dont over architect, and make solution to complex to standup) , present architecture for less technical audience
18. Any security concerns?
     rest: authenticate (where in code logic) BasicAuth, Session
19. No UI 

20. output file must 
batch all under max within 2 min period. 
max batch size is 1 per day, 100,000 per batch

at least one file per day
1 user in an hour, then another hour another user then 1 
create a file every 2 min (1min to respond)
0 users = 0 files (no need to account for this)

19. Any audit trails or logging needed?

22. if ftp endpoint is not reachable then or permission (then log)

can send email if more questions arise.


total in batch (across all users in batch) 

no delimiter between users
delimiter =  between users and the total


validation: 
-----------
 bankName has valid cardinality
 percent allocation must equal 100

 xml
 ----
 use xml parser lib

 deployment 
 ----------
 local deployment 

 design doc for peers
 -------------------
 seq
 uml
 network flow block diagram
 class diagram
 use case
 design pattern explination
 describe system requirements use M3, etc memory and processing

 whats the projected change. 
 Input:  types should support other kinds of asset accounts
 Output: output format may change to json, xml
         will want to support other aggregration output and filters aggregation 
           such: avg/bath, sum of batch of savingsAccount

Within a week, email to doug and george
explain why certain choices 

Steps to Solve:
---------------
step 1: beatuify the xml (easier to read)
step 2: convert to json (easier to see as object oriented)
step 4: make uml entity relationship diagram
	4.a: identify strong entites: User
    4.b: identify commonalities
       bankAccount, brokerageAccount, retirementAccount are all types of assets
       bankAccount, brokerageAccount, retirementAccount all have accountNumber (STRING) field
       bankAccount, brokerageAccount, retirementAccount all have amount (NUMBER) field
       bankAccount, brokerageAccount, retirementAccount all have description (OBJECT) field
    4.c: identity differences: 
    		bankAccount.description has bankAccountType field
    		bankAccount.description has bankName field
    		brokerageAccount.description has percent fields
    		retirementAccount.description has retirementAccountType fields
    4.d: identify relationships and cardinality, and primary keys
          User has 0 or 1 AssetProfile (renamed to makes sense given cardinality)
          AssetProfile has 1 AssetDescription
          AssetProfile has 0..n BankAccount
          AssetProfile has 0..1 RetirementAccount
          AssetProfile has 0..1 
    4.e: identify inheritance, compositions, and aggregations
         Inheritance: bankAccount, brokerageAccount, retirementAccount all can extend AssetAccount and have different Description Types
         Composition: 
         	- AssetAccount is Composed of AssetDescription (AssetDescription cannot exist without Asset)
         	- AssetAccount is Composed of AssetDescription (AssetDescription cannot exist without Asset)
         Aggregation: BankAccount could be considered an Aggregation to AssetProfile if it is seen as a Composition of User 


JSON
-----
userPayload = {
    "userId": "3562662382393",
    "assets": {
        "bankAccount": [
            {
                "accountNumber": "123895769767",
                "amount": "27576",
                "description": {
                    "bankAccountType": "CHECKING",
                    "bankName": "Chase"
                }
            },
            {
                "accountNumber": "123895769767",
                "amount": "12546",
                "description": {
                    "bankAccountType": "SAVINGS",
                    "bankName": "Citi"
                }
            }
        ],
        "brokerageAccount": {
            "accountNumber": "5556876547",
            "amount": "335464",
            "description": {
                "percentStocks": "50",
                "percentBonds": "30",
                "percentOther": "20"
            }
        },
        "retirementAccount": {
            "accountNumber": "88876758798",
            "amount": "6615",
            "description": {
                "retirementAccountType": "401k"
            }
        }
    }
}


Entity Diagram of Input. (I find it is best to always create a class structure that exactly matches the external systems' structure, but that does not mean we should store it that way)

UML DIAGRAM
--------                                                                               +-------------------------------+                                                     
                                        +---------------------------+             0..n |       AssetAccount            |                
+--------------------+             0..1 |   AssetProfile            |                  | ------------------------------|     
|       User         | 1                | ------------------------- |                  | PK accountNumber      (string)|              
| ------------------ |                  | PK assetProfileId (string)|-|--------------o<| FK assetProfileId     (string)|               
| PK userId  (string)|-|---------------o| FK userId         (string)|           +----|-| FK assetDescriptionId (string)|
|                    |                  |                           |           |      |     amount (decimal )         |  
+--------------------+                  |                           |           |     1|                               |         
                                        +---------------------------+           |      +-------------------------------+
                                                                                |      
                                                                                |
                                                                                |      +----------------------------------+ 
                                                                                | 0..1 |       AssetDescription           | 
                                                                                |      | -------------------------------- | 
                                                                                +----|<| PK assetDescriptionId    (string)| 
                                                                                       |    bankAccountType       (string)| * cardinality = CHECKING | SAVINGS
                                                                                       |    retirementAccountType (string)| * cardinality = 401k | IRA
                                                                                       |    bankName              (string)| 
                                                                                       |                                  | 
                                                                                       +----------------------------------+

Class Structure
---------------

public class User extends Serializable {
	private String userId;
	private AssetProfile asset;

	// getters & setters
}

public class AssetProfile extends Serializable {
	private AssetAccount[] bankAccount;
	private AssetAccount brokerageAccount;
	private AssetAccount retirementAccount;

	// getters & setters
}

public class AssetAccount extends Serializable {
	private String accountNumber;
	private 
	private Double amount 

	// getters & setters
}

public class AssetDescription extends Serializable {
	private String accountNumber;
	private Double amount

	// getters & setters
}

Database Modeling
-----------------

    1. For inheritence choose Table-Per-Type (TPT), Table-Per-Hierarchy (TPH), or Table-Per-Concrete (TPC)
          - I would choose Table-Per-Hierarchy (TPH) for the AssetAccount/AssetDescription

    2. Choose what are the real and derived entities (imho, database modeling should have real life modeling parallels)
       It looks like AssetProfile is derived and the asset accounts are really realated to the user, so lets do a model if we didnt save it. 
                                                                                             
DATABASE UML DIAGRAM (if persistence is required)
-------------------
ALT 1:
                                                                                +----------------------------------------+                                        
                                      +-------------------------------+    0..1 |       ASSET_ACCOUNT_DETAILS            |    
+-----------------------+        0..n |       ASSET_ACCOUNTS          |         | -------------------------------------- | 
|       USERS           | 1           | ------------------------------|         | PK asset_account_detail_id  VARCHAR(80)|                                         
| ----------------------|             | PK account_number  VARCHAR(80)|-------|<| PK account_number           VARCHAR(80)|                                                 
| PK user_id VARCHAR(80)|-|---------o<| FK user_id         VARCHAR(80)|         |    bank_account_type        VARCHAR(20)|                                      
|                       |             |                               |         |    retirement_account_type  VARCHAR(50)|
+-----------------------+             |                               |         |    bank_name               VARCHAR(255)|                                       
                                      +-------------------------------+         |    percent_stocks           NUMBER(3,2)| 
                                                                                |    percent_bonds            NUMBER(3,2)|
                                                                                |    percent_other            NUMBER(3,2)|
                                                                                +----------------------------------------+

ALT 2:
                                                                                +----------------------------------------+                                        
                                      +-------------------------------+    0..1 |       ASSET_ACCOUNT_DETAILS            |    
+-----------------------+        0..n |       ASSET_ACCOUNTS          |         | -------------------------------------- | 
|       USERS           | 1           | ------------------------------|    +--o<| PK asset_account_detail_id  VARCHAR(80)|                                         
| ----------------------|             | PK account_number  VARCHAR(80)|    |    |    bank_account_type        VARCHAR(20)|                                     
| PK user_id VARCHAR(80)|-|---------o<| FK user_id         VARCHAR(80)|    |    |    retirement_account_type  VARCHAR(50)|                                    
|                       |             | FK asset_account_detail_id    |-|--+    |    bank_name               VARCHAR(255)|
+-----------------------+             |    amount         NUMBER(10,0)|         |    percent_stocks           NUMBER(3,2)|                                       
                                      +-------------------------------+         |    percent_bonds            NUMBER(3,2)|
                                                                                |    percent_other            NUMBER(3,2)|
                                                                                +----------------------------------------+

ALT 3: (my preference)
                                                                                 +----------------------------------------+                                        
                                      +--------------------------------+    0..1 |      ASSET_ACCOUNT_ALLOCATIONS         |    
+-----------------------+        0..n |       ASSET_ACCOUNTS           |         | -------------------------------------- | 
|       USERS           | 1           | ------------------------------ |    +--|<| PK asset_account_id         VARCHAR(80)|                                         
| ----------------------|             | PK asset_account_id VARCHAR(80)|-|-/     |    stocks                   NUMBER(3,2)|                                     
| PK user_id VARCHAR(80)|-|---------o<| FK user_id          VARCHAR(80)|         |    bonds                    NUMBER(3,2)|                                   
|    external_user_id   |             |    bank_name       VARCHAR(255)|         |    other                    NUMBER(3,2)| 
+-----------------------+             |    account_number   VARCHAR(80)|         +----------------------------------------+
									  |    amount          NUMBER(10,2)|                                              
                                      |    account_type     VARCHAR(20)|         
                                      |    tax_code          VARCHAR(4)|         
                                      +--------------------------------+ 


Mapping

Map input class structure to output class structure or file format.

382201-27576-335464-12546-6615=0, looks like total uses amount from all accounts

                                                                                             
