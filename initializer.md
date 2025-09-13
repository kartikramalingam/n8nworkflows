I want you to srap the existing logic of initializer node . The logic of the initializer should be simple . 

1. Define the default values at the top of the program ( similar to this )
   
   ```
      'asofdate': None,  # None => today
      'securitytype': ["Kartik","Akira"],
      'marketSector': ["Sandhya", "Atharva"],
      'numberofdays': 20,
      'marketcaplowerlimit': 250000,
      'sharefloat': 0.7,
   
      {    "market": {
        "filters": {},
        "groups": [
          {
            "asOf": "2026-05-06",
            "attribute_code": "marketSector",
            "value": {
              "eq": "Equity",
              "type": "text"
            }
          }
        ],
        "limit": 100,
        "logic": "OR"
      }
   
      }, 
   
      {    "equity": {
        "filters": {
          "company_ids": []
        },
        "groups": [
          {
            "asOf": "2026-05-06",
            "attribute_code": "securitytype",
            "value": {
              "eq": "ETF",
              "type": "text"
            }
          }
        ],
        "limit": 100,
        "logic": "AND"
      }
   
      }
   ```

```
2. Do not do any processing.

3. if these valeus are provided in the webhook API post body , then override  the default value provided in the webhook request
```


