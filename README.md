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

# Issue 1
There is an issue. That is converting `pickup_location` or `dropoff_location`. If the source value can not be converted to `float` and the destination value could be abnomal or If latitude or logtitude is greater than or equal 180 or slower than or euqual -180, errors will occur. 

```
[2019-11-05T07:31:23,515][WARN ][logstash.outputs.elasticsearch] Could not index event to Elasticsearch. {:status=>400, :action=>["index", {:_id=>nil, :_index=>"nyc_taxi_data_2014", :_type=>"doc", :routing=>nil}, #<LogStash::Event:0xe45b26>], :response=>{"index"=>{"_index"=>"nyc_taxi_data_2014", "_type"=>"doc", "_id"=>"VO95Om4B6PHSiacoribN", "status"=>400, "error"=>{"type"=>"mapper_parsing_exception", "reason"=>"failed to parse field [dropoff_location] of type [geo_point]", "caused_by"=>{"type"=>"parse_exception", "reason"=>"latitude must be a number"}}}}}
```

```
[2019-11-05T08:41:25,839][WARN ][logstash.outputs.elasticsearch] Could not index event to Elasticsearch. {:status=>400, :action=>["index", {:_id=>nil, :_index=>"nyc_taxi_data_2014", :_type=>"doc", :routing=>nil}, #<LogStash::Event:0xb2f94a0>], :response=>{"index"=>{"_index"=>"nyc_taxi_data_2014", "_type"=>"doc", "_id"=>"6Ta5Om4B6PHSiacozogw", "status"=>400, "error"=>{"type"=>"mapper_parsing_exception", "reason"=>"failed to parse field [dropoff_location] of type [geo_point]", "caused_by"=>{"type"=>"illegal_argument_exception", "reason"=>"illegal longitude value [-775.416665] for dropoff_location"}}}}}
```

I tried to edit elasticsearch index to be toleranced with abnormal data:

```
"dropoff_location":{ 
	"type":"geo_point",
	"ignore_malformed": true
}
```

The issue still is not solved.

Also, I tried to use if condition in logstash configuration to replace all abnomal latitude or longitude to zero. But in the case that source value can not be converted to `float`, an other error occured. 

I found out that `ruby filter` would solve this issue, but because of lacking time, I could not try.

In conclustion, this issue still has not been solved.

Because of this issue, the data is missed 263 records.

# Issue 2
Because of the too big volumn of data, I could not add script field that equal difference between `pickup_datetime` and `dropoff_datetime`. This action would accupy a lot of computation resource.

# Automation with concourse
When I started doing this project, It was first time I heared about CI/CD processing in software development. Therefore I can not apply it. I think applying it take me about 2 weeks or more instead of 5 days, 1 week for understanding about CI/CD and get familiar with Ansible (an similar framwork with Concourse but more popular, getting familiar with a more popular framwork is often easier) and 1 week for starting with Concourse.