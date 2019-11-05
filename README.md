# Data
nyc_taxi_data_2014 contains only one table with the following fields: 
- vendor_id: category
- pickup_datetime: date
- dropoff_datetime: date 
- passenger_count: numeric/category
- trip_distance: numeric 
- pickup_longitude: numeric/geo_point 
- pickup_latitude: numeric/geo_point 
- rate_code: numeric/category
- store_and_fwd_flag: boolean 
- dropoff_longitude: numeric/geo_point 
- dropoff_latitude: numeric/geo_point 
- payment_type: category
- fare_amount: numeric
- surcharge: numeric 
- mta_tax: numeric/category
- tip_amount: numeric 
- tolls_amount: numeric
- total_amount numeric

# Data modeling
Elasticsearch is a NoSQL database and we can not execute ad-hoc queries to it, so data model must adapt to bussiness requirements. Howerver because of the simplicity of data struct, there are not much problem in data modeling needed to consider.

# Elasticsearch 

###Index mapping 
I have to specify index before running logstash to put data because I have not found out any way to map location to geo_point data type automatically.
```
"mappings":{ 
      "doc":{ 
         "properties":{ 
            "dropoff_datetime":{ 
               "type":"date"
            },
            "dropoff_location":{ 
               "type":"geo_point"
            },
            "fare_amount":{ 
               "type":"float"
            },
            "mta_tax":{ 
               "type":"float"
            },
            "passenger_count":{ 
               "type":"long"
            },
            "payment_type":{ 
               "type":"text",
               "fields":{ 
                  "keyword":{ 
                     "type":"keyword",
                     "ignore_above":256
                  }
               }
            },
            "pickup_datetime":{ 
               "type":"date"
            },
            "pickup_location":{ 
               "type":"geo_point"
            },
            "rate_code":{ 
               "type":"long"
            },
            "store_and_fwd_flag":{ 
               "type":"boolean"
            },
            "surcharge":{ 
               "type":"float"
            },
            "tip_amount":{ 
               "type":"float"
            },
            "tolls_amount":{ 
               "type":"float"
            },
            "total_amount":{ 
               "type":"float"
            },
            "trip_distance":{ 
               "type":"float"
            },
            "vendor_id":{ 
               "type":"text",
               "fields":{ 
                  "keyword":{ 
                     "type":"keyword",
                     "ignore_above":256
                  }
               }
            }
         }
      }
   }
```

# Logstash 

### Configuration
In the following code, I just want to run logstash one time to put the data to elasticsearch and I will turn off logstash service after that. Therefore, I added the line: `sincedb_path => "/dev/null"`.

```
input { 
	file { 
		path => "~/nyc_taxi_data_2014-Data-Engineering/nyc_taxi_data_2014_short_sample.csv" 
		start_position => "beginning" 
		sincedb_path => "/dev/null"
	} 
}

flter {
	csv {
        columns => ["vendor_id", "pickup_datetime", "dropoff_datetime", "passenger_count", "trip_distance", "pickup_longitude", "pickup_latitude", "rate_code", "store_and_fwd_flag", "dropoff_longitude", "dropoff_latitude", "payment_type", "fare_amount", "surcharge", "mta_tax", "tip_amount", "tolls_amount", "total_amount"]
        separator => ","
        skip_header => "true"
    }
    date {
		match => [ "pickup_datetime", "yyyy-MM-dd HH:mm:ss" ]
		target => "pickup_datetime"
	}
	date {
		match => [ "dropoff_datetime", "yyyy-MM-dd HH:mm:ss" ]
		target => "dropoff_datetime"
	}
	mutate {
		convert => {
			"passenger_count" => "integer"
			"trip_distance" => "float"
			"rate_code" => "integer"
			"store_and_fwd_flag" => "boolean"
			"fare_amount" => "float"
			"surcharge" => "float"
			"mta_tax" => "float"
			"tip_amount" => "float"
			"tolls_amount" => "float"
			"total_amount" => "float"
		}
		rename => {
			"pickup_longitude" => "[pickup_location][lon]"
			"pickup_latitude" => "[pickup_location][lat]"
		}
		convert => {
			"[pickup_location][lon]" => "float"
			"[pickup_location][lat]" => "float"
		}
		rename => {
			"dropoff_longitude" => "[dropoff_location][lon]"
			"dropoff_latitude" => "[dropoff_location][lat]"
		}
		convert => {
			"[dropoff_location][lon]" => "float"
			"[dropoff_location][lat]" => "float"
		}
	}
}


output { 
	elasticsearch { 
		host => "localhost:9200" 
		index => "nyc_taxi_data_2014" 
	}
	stdout {}
}
```

# Kibana
Url: http://13.251.89.102
To login nginx server, please input username and password:
Username: kibanaadmin
Password: LcBZD2fesTa5

# Automation with concourse
When I started doing this project, It was first time I heared about CI/CD processing in software development. Therefore I can not apply it. I think applying it take me about 2 weeks or more, 1 week for understanding about CI/CD and get familiar with Ansible (an similar framwork with Concourse but more popular, getting familiar with a more popular framwork is often easier) and 1 week for starting with Concourse.