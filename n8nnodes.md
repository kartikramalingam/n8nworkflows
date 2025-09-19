Create a n8n workflow with the following features : 

1. The workflow should have following nodes.
   
   1. The webhook should have post webhook where the parameters are provided in a json object. The node should be called as "input trigger" . Parameters to be included for the first built :
      
      1. securitytype - list of securitytype ( text) example : 'Common Stock', 'ETF'
      
      2. marketSector - list of  marketsector( text ) example : 'Equity', 'Debt'
      
      3. numberofdays - numeric
      
      4. marketcaplowerlimit - float
      
      5. sharefloat - float
      
      6. asofdate - date
      
      7. daterange - daterange 
      
      8. operator - string  such as 'OR', 'AND'
   
   2. The second node should be a python code , which takes the parameters from above node and sets a default value if the parameter is not provided. The node should be called "Initializer". This node should get data from node1 
   
   3. The third node should be an http request .  This node should get data from node 2
      
      1. populate "logic" with operator from node 2
      
      2. for the first group , populate "attribute_code"  with securitytype from node 2, populate "asOf" with asofdate from node 2 
      
      3. for the first group , populate "attribute_code" with marketSector from node 2, populate "asOf" with asofdate from node 2 ,
   
   ```
   curl --request POST \
     --url http://192.168.0.59:8000/nav/search \
     --header 'content-type: application/json' \
     --data '{
     "logic": "OR",
     "groups": [
       {
         "attribute_code": "figi",
         "asOf": "2026-05-06",
         "value": {
           "type": "text",
           "eq": "BBG01293F546"
         }
       },
       {
         "attribute_code": "figi",
         "asOf": "2026-05-06",
         "value": {
           "type": "text",
           "regex": "[A-B]BG01293F5P*"
         }
       }
       }
     ],
     "filters": {
       "company_ids": [],
       "node_type_codes": [],
       "source": null
     },
     "sort": [
       {
         "field": "valid_from",
         "dir": "asc"
       },
       {
         "field": "node_inst_id",
         "dir": "asc"
       }
     ],
     "limit": 100
   }'
   ```
   
   1. The fourth  node should be an http request .  This node should get data from node 2
      
      1. populate "logic" with operator from node 2
      
      2. for the first group , populate "attribute_code"  with securitytype from node 2, populate "asOf" with asofdate from node 2 
      
      3. for the first group , populate "attribute_code" with marketSector from node 2, populate "asOf" with asofdate from node 2 ,
      
      ```
      curl --request POST \
        --url http://192.168.0.59:8000/nav/search \
        --header 'content-type: application/json' \
        --data '{
        "logic": "OR",
        "groups": [
          {
            "attribute_code": "figi",
            "asOf": "2026-05-06",
            "value": {
              "type": "text",
              "eq": "BBG01293F546"
            }
          },
          {
            "attribute_code": "figi",
            "asOf": "2026-05-06",
            "value": {
              "type": "text",
              "regex": "[A-B]BG01293F5P*"
            }
          }
          }
        ],
        "filters": {
          "company_ids": [],
          "node_type_codes": [],
          "source": null
        },
        "sort": [
          {
            "field": "valid_from",
            "dir": "asc"
          },
          {
            "field": "node_inst_id",
            "dir": "asc"
          }
        ],
        "limit": 100
      }'
      ```
