AI ChatBot with GPT-4 & MindsDB Code 

-- Generate youtube credentials file 
-- connect youtube with this credentials file to mindsdb
CREATE DATABASE mindsdb_youtube
WITH ENGINE = 'youtube',
PARAMETERS = {
    "credentials_file": "/mnt/desktop/yt_credentials.json"
};



--- Connection success
--- You can list all the linked databases using the command below:
SHOW DATABASES WHERE name = 'mindsdb_youtube';

SELECT *
FROM --define a table name to preview the data
LIMIT 10; 



CREATE ML_ENGINE openai_engine 
    FROM openai 
    USING 
        open_api_key ='REDACTED (API token key goes here)'

CREATE MODEL yt_sentiment_classifier
PREDICT sentiment
USING
 engine = 'openai_engine',
 model_name = 'gpt-3.5-turbo',
 prompt_template = 'describe the sentiment of the comments
                        strictly as "positive", "spam", or "negative".
                        "I love this video":positive 
                        "It is not a helpful content" :negative
                        "{{comment}}.":';

SELECT *
FROM mindsdb_youtube.comments AS c
JOIN yt_sentiment_classifier AS m
WHERE c.video_id = "REDACTED (youtube video ID goes here)";



CREATE MODEL yt_reply_model 
PREDICT reply 
USING 
    engine = 'openai_engine',
    model_name= 'gpt-3.5-turbo',
    prompt_template = 'briefly respond to the youtube comment "{{comment}}";
                        as a context, use the video transcript "{{transcript}}"';



SELECT comment_id, comment 
FROM mindsdb_youtube.comments 
WHERE video_id = "REDACTED (youtube video ID goes here)";


SELECT c.comment_id AS comment_id, 
        c.comment AS comment,
        m.sentiment AS sentiment 
FROM mindsdb_youtube.comments AS c 
JOIN yt_sentiment_classifier AS m 
WHERE c.video_id = "REDACTED (youtube video ID goes here)"
AND c.comment_id = "REDACTED (comment ID goes here)"



SELECT c.comment_id AS comment_id, 
    c.comment AS comment, 
    m1.sentiment AS sentiment, 
    m2.reply AS reply 
FROM mindsdb_youtube.comments c 
LEFT JOIN mindsdb_youtube.videos v 
ON c.video_id = v.video_id 
JOIN yt_sentiment_classifier AS m1 
JOIN yt_reply_model AS m2 
WHERE c.video_id = "REDACTED ()"
AND v.video_id = "REDACTED (youtube video ID goes here)"
AND c.comment_id = "REDACTED (comment ID goes here)"



INSERT INTO mindsdb_youtube.comments (comment_id, reply)
    SELECT c.comment_id AS comment_id, 
        m2.reply AS reply 
    FROM mindsdb_youtube.comments c 
    LEFT JOIN mindsdb_youtube.videos v 
    ON c.video_id = v.video_id 
    JOIN yt_sentiment_classifier AS m1 
    JOIN yt_reply_model AS m2 
    WHERE c.video_id = "REDACTED (youtube video ID goes here)"
    AND v.video_id = "REDACTED (youtube video ID goes here)"
    AND c.published_at > LAST 
    AND m1.sentiment = 'positive';



CREATE DATABASE yt_slack 
WITH 
    ENGINE = 'slack'
    PARAMETERS = {
        "token": "REDACTED (slack bot token goes here )"
    }



SELECT * 
FROM yt_slack.channels 
WHERE channel = "mindsdb-test";



INSERT INTO yt_slack.channels (channel, text)
VALUES("mindsdb-test", "Hai")



INSERT INTO yt_slack.channels (channel, text)
    SELECT "mindsdb-test" AS channel, 
        concat('Video ID: ', c.video_id, chr(10),
                'Comment ID: ', c.comment_id, chr(10), 
                'Author: ', c.display_name, chr(10), 
                'Comment: ', c.comment, chr(10), 
                'Sentiment: ', m.sentiment, chr(10),
                'Sample reply: ', m2.reply) AS text
    FROM mindsdb_youtube.comments c
    LEFT JOIN mindsdb_youtube.videos v 
    ON c.video_id = v.video_id 
    JOIN yt_sentiment_classifier AS m
    JOIN yt_reply_model AS m2
    WHERE c.video_id = "REDACTED (youtube video id goes here)"
    AND v.video_id = "REDACTED (youtube video id goes here)"
    AND c.published_at > LAST
    AND m.sentiment = 'negative'; 



CREATE DATABASE psql_datasource 
WITH ENGINE = 'postgres',
PARAMETERS = {
    "host": "ep-royal-wildflower-a4wdnc68.us-east-1.aws.neon.tech",
    "database": "pgdb",
    "schema": "public",
    "user": "pgdb_owner",
    "password": "9sIyfXqDO1Uj"
};



CREATE JOB youtube_chatbot (
    -- save the recently added comments into a table 
    CREATE OR REPLACE TABLE psql_datasource.recent_comments (
        SELECT * 
        FROM mindsdb_youtube.comments c 
        LEFT JOIN mindsdb_youtube.videos v 
        ON c.video_id = v.video_id 
        WHERE c.video_id = "REDACTED (youtube video ID goes here)"
        AND v.video_id = "REDACTED (youtube video ID goes here)"
        AND c.published_at > LAST 
        )
    );



INSERT INTO mindsdb_youtube.comments (comment_id, reply)
        SELECT c.comment_id AS comment_id, 
               m2.reply AS reply 
        FROM psql_datasource.recent_comments AS c 
        JOIN yt_sentiment_classifier AS m 
        JOIN yt_reply_model AS m2 
        WHERE m.sentiment = 'positive';

        

INSERT INTO yt_slack.channels (channel, text)
    SELECT "mindsdb-test" AS channel, 
        concat('Video ID: ', c.video_id, char(10),
        'Comment ID: ', c.comment_id, chr(10),
        'Author: ', c.display_name, chr(10),
        'Comment: ', c.comment, chr(10),
        'Sentiment: ', m.sentiment, chr(10),
        'Sample reply: ', m2.reply) AS text
    FROM psql_datasource.recent_comments AS c 
    JOIN yt_sentiment_classifier AS m 
    JOIN yt_reply_model AS m2 
    WHERE m.sentiment = 'negative';
)
EVERY 10 minutes;