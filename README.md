# AWS Weather Alert System using Lambda, SNS, and EventBridge

This project automatically checks the current weather conditions (temperature, humidity, wind speed, and sky condition) for a specific city using the OpenWeatherMap API, analyzes the data, and sends weather alerts via Amazon SNS (email or SMS).

It runs automatically at scheduled intervals using Amazon EventBridge.

### Project Overview
#### How it works

__EventBridge Rule (Scheduler)__ triggers the __AWS Lambda function__ every few hours (for example, every 3hour).

__Lambda__ calls the __OpenWeatherMap API__ to get the latest weather data for your chosen city (e.g., Pune).

The Lambda function analyzes the data:
* Checks temperature (hot/cold)
* Checks humidity (high/low)
* Checks wind speed (windy/storm)
* Checks weather condition (rain, snow, fog, etc.)

Based on the results, it prepares a __summary + alerts__ message.

The message is published to an __SNS Topic__, which notifies users via __Email or SMS__.

### AWS Services Used

* __AWS Lambda__ -->	Executes the Python script that fetches and processes weather data.

* __Amazon SNS (Simple Notification Service)__ -->	Sends alerts to users via email or SMS.

* __Amazon EventBridge (CloudWatch Events)__	--> Triggers Lambda automatically at scheduled intervals.

* __OpenWeatherMap API__	--> Provides real-time weather data.

### Lambda Function Workflow

1. Fetch weather data using OpenWeatherMap API

2. Extract temperature, humidity, wind speed, and conditions

3. Analyze values to detect:

* 🔥 Extreme heat

* ❄️ Cold weather

* 🌪️ Strong winds

* 🌧️ Rain / storm / fog

4. Combine everything into a detailed report

5. Send results to SNS for user notification

### Environment Variables in Lambda

Set these environment variables in your Lambda configuration:


| Variable Name         | Description                      | Example                                          |
| --------------------- | -------------------------------- | ------------------------------------------------ |
| `OPENWEATHER_API_KEY` | Your API key from OpenWeatherMap | `YOUR_WEATHER_API`               |
| `SNS_TOPIC_ARN`       | The ARN of your SNS topic        | `YOUR_SNS_ARN` |                                          |

![](./images/Screenshot%20(83).png)

### Lambda Function Code (Python)
```python
import json
import os
import requests
import boto3

def lambda_handler(event, context):
    city = os.environ.get('CITY', 'Pune')
    api_key = os.environ['OPENWEATHER_API_KEY']
    sns_topic_arn = os.environ['SNS_TOPIC_ARN']

    url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&units=metric&appid={api_key}"
    response = requests.get(url)
    data = response.json()

    temp = data['main']['temp']
    feels_like = data['main']['feels_like']
    humidity = data['main']['humidity']
    wind_speed = data['wind']['speed'] * 3.6  # m/s → km/h
    condition = data['weather'][0]['description'].lower()

    sns = boto3.client('sns')
    alerts = []

    # Temperature Alerts
    if temp > 38:
        alerts.append(f"🔥 Extreme Heat Alert! It's {temp}°C in {city}. Avoid going outside.")
    elif 35 < temp <= 38:
        alerts.append(f"🥵 High Temperature Alert! It's {temp}°C. Stay hydrated!")
    elif temp < 5:
        alerts.append(f"❄️ Extreme Cold Alert! It's {temp}°C in {city}. Bundle up!")
    elif 5 <= temp < 10:
        alerts.append(f"🥶 Cold Weather Alert! It's {temp}°C in {city}. Dress warmly!")

    # Humidity Alerts
    if humidity > 85:
        alerts.append(f"💧 High Humidity Alert! Humidity is {humidity}% — air feels heavy.")
    elif humidity < 25:
        alerts.append(f"🌵 Low Humidity Alert! Humidity is {humidity}%. Skin may feel dry.")

    # Wind Alerts
    if wind_speed > 40:
        alerts.append(f"🌪️ Strong Wind Alert! Wind speed is {wind_speed:.1f} km/h. Secure loose objects.")
    elif 25 < wind_speed <= 40:
        alerts.append(f"💨 Windy Conditions! Wind speed: {wind_speed:.1f} km/h.")

    # Condition Alerts
    if "rain" in condition:
        alerts.append(f"🌧️ Rain Alert! {condition.title()} in {city}. Carry an umbrella.")
    elif "drizzle" in condition:
        alerts.append(f"☔ Drizzle Alert! Light rain expected in {city}.")
    elif "thunder" in condition or "storm" in condition:
        alerts.append(f"⛈️ Storm Alert! {condition.title()} in {city}. Stay indoors.")
    elif "snow" in condition:
        alerts.append(f"❄️ Snow Alert! {condition.title()} in {city}. Roads may be slippery.")
    elif "fog" in condition or "mist" in condition or "haze" in condition or "smoke" in condition:
        alerts.append(f"🌫️ Low Visibility Alert! {condition.title()} in {city}. Drive carefully.")
    elif "clear" in condition:
        alerts.append(f"☀️ Clear Sky! Enjoy the good weather in {city}.")
    elif "cloud" in condition:
        alerts.append(f"☁️ Cloudy Skies! Expect mild conditions in {city}.")

    # Summary
    summary = (
        f"📍 Weather Summary for {city}\n"
        f"🌡️ Temperature: {temp}°C (Feels like {feels_like}°C)\n"
        f"💧 Humidity: {humidity}%\n"
        f"🌬️ Wind Speed: {wind_speed:.1f} km/h\n"
        f"🌤️ Condition: {condition.title()}"
    )

    if alerts:
        message = summary + "\n\n🚨 Alerts:\n" + "\n".join(alerts)
        subject = f"⚠️ Weather Alert for {city}"
    else:
        message = summary + "\n\n✅ No severe weather conditions detected."
        subject = f"✅ Weather Update for {city}"

    sns.publish(TopicArn=sns_topic_arn, Message=message, Subject=subject)
    print("Notification sent successfully!")

    return {'statusCode': 200, 'body': json.dumps('Weather check completed successfully!')}
```

### Create SNS Topic and Subscription

1. Go to ___Amazon SNS → Topics → Create topic__

   * Type: Standard

   * Name: `lambdaTopic`

2. Click __Create Subscription__

   * Protocol: `Email`

   * Endpoint: your email address

   * Confirm the subscription by clicking the link in your email.

### Create EventBridge Rule (Scheduler)

1. Go to EventBridge → Rules → Create rule

2. Choose Schedule

3. Set your schedule (e.g. `rate(3 hour)` or every day at 8 AM using `cron(0 8 * * ? *)`)

4. Under Target, choose:

 ``` 
 AWS service → Lambda function 
 ```


Then select your Lambda function.

5. Create the rule.

### Test the Lambda Function

1. Open your Lambda function.

2. Click __Test → Configure test event → Create new test event__.

3. Run the __test__.

4. Check your __email__ (from SNS) for the weather alert.

### Example Output (Email)
```
📍 Weather Summary for Pune
🌡️ Temperature: 30°C (Feels like 32°C)
💧 Humidity: 70%
🌬️ Wind Speed: 12.6 km/h
🌤️ Condition: Light Rain

🚨 Alerts:
🌧️ Rain Alert! Light Rain in Pune. Carry an umbrella.

```
![](./images/Screenshot%20(80).png)

### Future Improvements

* Support multiple cities at once

* Add weather trend analysis (day-to-day comparison)

* Store logs in DynamoDB

### Acknowledgements

* AWS Lambda, SNS, and EventBridge for automation

* OpenWeatherMap for free weather data API