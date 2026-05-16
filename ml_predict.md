# Flink ML_PREDICT 

Limited to the Ververica Platform and OpenAI API  

## Source table example:
Use your own table source or enriched data  

```
CREATE TABLE customer_reviews (
  user_name    STRING,
  star_rating  INT,
  tone_idx     INT,
  body_idx     INT,
  user_comment AS CONCAT(
    CASE tone_idx
      WHEN 0 THEN 'Honestly, '
      WHEN 1 THEN 'Wow — '
      WHEN 2 THEN 'Not gonna lie, '
      WHEN 3 THEN 'Meh. '
    END,
    CASE body_idx
      WHEN 0 THEN 'this exceeded every expectation I had.'
      WHEN 1 THEN 'it stopped working after two uses.'
      WHEN 2 THEN 'I am not sure how I feel about it yet.'
      WHEN 3 THEN 'the build quality is genuinely impressive but the price stings.'
      WHEN 4 THEN 'customer service refused to help with a clear defect.'
      WHEN 5 THEN 'it does the job and nothing more.'
    END
  ),
  generated_at AS CURRENT_TIMESTAMP
) WITH (
  'connector' = 'datagen',
  'rows-per-second' = '2',
  'fields.user_name.length' = '12',
  'fields.star_rating.min' = '1',
  'fields.star_rating.max' = '5',
  'fields.tone_idx.min' = '0',
  'fields.tone_idx.max' = '3',
  'fields.body_idx.min' = '0',
  'fields.body_idx.max' = '5'
);
```

## Sink table example
Use your own target storage (table, topic, object storage, etc) for the output  

```
CREATE TABLE enriched_reviews_kafka (
  user_name       STRING,
  star_rating     INT,
  user_comment    STRING,
  sentiment       STRING,
  generated_at    TIMESTAMP_LTZ(3),
  synced_at       TIMESTAMP_LTZ(3),
  latency_ms      BIGINT
) WITH (
  'connector' = 'kafka',
  'topic'     = 'enriched_reviews',
  'properties.bootstrap.servers' = 'host.minikube.internal:9092',
  'format' = 'json',
  'json.timestamp-format.standard' = 'ISO-8601'
);
```

## Define the Model
Replace the <token> with the proper value. Be cautious with the rate of API calls to avoid high costs.

```
CREATE MODEL ai_analyze_sentiment
INPUT  (`input`  STRING)
OUTPUT (`output` STRING)
WITH (
  'provider'      = 'openai',
  'task'          = 'classification',
  'api-key'       = '<token>',
  'model-version' = 'gpt-4o-mini',
  'prompt' = 'Classify the customer review into exactly one label: positive, negative, neutral, or mixed. Respond with only the lowercase label, no punctuation.'
);
```

## Flink SQL job
This job adds column `generated_at`, `synced_at` and `latency_ms` to measure pipeline latency.

```
INSERT INTO enriched_reviews_kafka
SELECT
  user_name,
  star_rating,
  user_comment,
  `output`                                                       AS sentiment,
  generated_at,
  CURRENT_TIMESTAMP                                              AS synced_at,
  (CAST(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP) * 1000 AS BIGINT) + EXTRACT(MILLISECOND FROM CURRENT_TIMESTAMP)) - 
  (CAST(EXTRACT(EPOCH FROM generated_at) * 1000 AS BIGINT) + EXTRACT(MILLISECOND FROM generated_at))    AS latency_ms 
FROM ML_PREDICT(
  INPUT => TABLE customer_reviews,
  MODEL => MODEL ai_analyze_sentiment,
  ARGS  => DESCRIPTOR(user_comment)
);
```


