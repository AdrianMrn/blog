# How I used Google Cloud Functions to survive a heatwave

[See the repo on GitHub](https://github.com/AdrianMrn/tado-temperature-notifier)

Disclaimer: this is the first real blogpost I've ever written. If you've got any storytelling tips or other advice, don't hesitate to let me know on Twitter @AdriaanMrn

This summer in Belgium has been a very hot one, and we're bound to have a couple more very warm weeks during the rest of summer.

I'm not the best at dealing with this kind of heat, and with the temperature inside hitting 28° C, I've been trying to get my home to cool down on a budget: I open the windows when the temperature outside is cooler than inside, and close them when it warms up. I've also noticed the sun hits the living room starting at around 7 in the morning, which means I should close the curtains right around that time to stop the tile floor from heating up.

From the start, I knew it was likely that I would forget to close the windows in time. I would need someone - or something - to remind me. I saw a talk about Google Cloud by Bram Van Damme at JSConf.be a while back, so I decided this could be a fun little project to play around with GCP.

These are the features I decided on for an MVP:

- When the temperature outside my apartment is lower than the temperature inside, tell me to open the windows, and the other way around
- When the sun could be heating up the living room, tell me to close the curtains

I'll start off by outlining the different moving parts to my little project:

## Google Cloud Functions

Google Cloud Functions allows you to upload a script that you can run whenever you want, without having to have your computer or a server running. It's basically Google's answer to AWS Lambda.

When you create a new function, you get a trigger URL. When sending a GET request to this URL, the cloud function executes.

Cloud Functions is free for the first 2 million executions per month, your first 400,000 GB-seconds (1 GB of RAM used for 1 second) and your first 200,000 GHz-seconds per month are also free. Finally, the free tier includes 5GB of internet egress traffic per month. This free tier does not expire after a year.

## Google Cloud Scheduler

Google Cloud Scheduler allows you to create actions that will start the script, using a cronjob. Here's the cronjob to make it run every 5 minutes:

```
*/5 * * * *
```

Google Cloud Scheduler is free for your first 3 actions per month, with an unlimited amount of runs for each action.

## Google Cloud Datastore

Google Cloud Datastore is a NoSQL database that's very well integrated with GCP Functions. A GCP Function that operates in the same project as a Datastore doesn't need any authentication settings to read and write to the database.

Reading and writing simple pieces of data, like what we're dealing with here, is made effortless by Datastore:

```js
// reading
const value = await datastore.get(datastore.key(["type", "key"]));

// writing/updating
datastore.upsert({ key: datastore.key(["type", "key"]), data: { foo: "bar" } });
```

## TypeScript

I decided to use TypeScript to write the script for two main reasons:

Firstly, I've gotten so used to writing TypeScript, that I usually end up writing code faster than when using JavaScript. Especially when including the time saved by avoiding the vast majority of runtime errors, and the time spent refactoring.

Secondly, I'll have to do some API calls in this project. Being able to take a response and generate a type from it makes it much easier to navigate the response by giving you autocompletion on all properties. It will also make it easier to come back to the project after a while to add some more features.

Using TypeScript may seem like extra work to get going, but with how robust the TypeScript compiler is nowadays, you barely need any extra configuration. My `start` command in `package.json` takes less than a second to compile and start running the script:

```json
"start": "tsc src/index.ts --outDir ./dist && node dist/index.js"
```

To deploy the script to Google Cloud, you just zip the output from the compilation step along with your `package.json` file and upload this zip file to your function in Google Cloud Functions. It's also possible to automate a part of this using the GCP CLI.

There's no need for any further optimizations during bundling like uglification or minification, since the code doesn't need to be downloaded by a client. The difference in setup time for Cloud Functions would be negligible.

I removed any TypeScript annotations for the simplified code examples I bring up in this post, to make them readable to everyone.

## Getting the temperature readings

To get started, I had to figure out how to accurately measure the outside and inside temperatures.

One option would be to use a Raspberry Pi with two temperature sensors. The biggest problem with this solution is that I don't have any temperature sensors, and if I ordered them from AliExpress, they probably wouldn't arrive before winter. It would also add a complexity layer to get the readings from the Pi to somewhere the function on Google Cloud could access them.

Luckily, it just so happens I got myself a Tado smart thermostat last year. Tado has a REST API you can use to read your data, in my case I'll be using it reading my living room's current temperature:

```js
const endpoint = `https://my.tado.com/api/v2/homes/${homeId}/zones/${zoneId}/state`;

const response = await Axios.get(endpoint, {
  headers: { Authorization: `Bearer ${tadoToken}` },
});

const indoorTemperature =
  response.data.sensorDataPoints.insideTemperature.celsius;
// 25.2
```

(simplified example, see [full code](https://github.com/AdrianMrn/tado-temperature-notifier/blob/master/src/function.ts))

While writing the script, I found out the Tado API also has an endpoint that returns the current outdoors temperature in my city. It also tells you the current type of weather, like whether it's sunny or overcast:

```js
const endpoint = `https://my.tado.com/api/v2/homes/${homeId}/weather`;

const response = await Axios.get(endpoint, {
  headers: { Authorization: `Bearer ${tadoToken}` },
});

const outsideTemperature = response.data.outsideTemperature.celsius;
// 23.8
const weatherState = response.data.weatherState.value;
// SUNNY
```

(simplified example, see [full code](https://github.com/AdrianMrn/tado-temperature-notifier/blob/master/src/function.ts))

Using all these values, the script could then decide whether the windows and curtains should be open or closed to cool down my home, or at least stop it from heating up more.

```js
const shouldOpenWindows = outsideTemperature < insideTemperature;
const shouldCloseCurtains = weatherState === "SUN";
```

I don't want to receive an email every time the script runs, so I need to compare these final values to the result of the last script run.

I used Google Cloud Datastore to save the values of `shouldOpenWindows` and `shouldCloseCurtains` on each run, and compare them to the values of the last run:

```js
import { Datastore } from "@google-cloud/datastore";

const datastore = new Datastore({ projectId: "my-project-id" });

// reading the old values
const oldValues = await datastore.get(datastore.key(["state", "oldValues"]));
// > [{ oldShouldOpenWindows: true, oldShouldCloseCurtains: true }]

// writing new values
const entity = {
  key: datastore.key(["state", "oldValues"]),
  data: { shouldOpenWindows, shouldCloseCurtains },
};

datastore.upsert(entity);
```

(simplified example, see [full code](https://github.com/AdrianMrn/tado-temperature-notifier/blob/master/src/datastore.ts))

When running a Cloud Functions script in the same project as Cloud Datastore, you don't need any extra authentication for read and write actions.

A notification should be sent out to me when the old and new values for `shouldOpenWindows` and/or `shouldCloseCurtains` are different from each other.

## Sending out notifications

I decided that the easiest way to send out notifications would be to send myself an email, I used the SendGrid API for this.

After SendGrid's trial period, you get 100 free email sends per day. To be completely safe, I could set the Cloud Scheduler cronjob to only run once 15 minutes. However, I don't expect the weather to be that inconsistent that the script needs to send me a notification every time the script runs, so I think I'll still be safe running it every 5 minutes.

I used the `@sendgrid/mail` SDK, which makes sending emails a breeze:

```js
const sgMail = require('@sendgrid/mail');
sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

sgMail.send({
	to: process.env.NOTIFY_EMAIL,
	from: process.env.NOTIFY_EMAIL,
	subject: 'Temperature update',
	text: 'Open your windows!',
	html: '<strong>Open your windows!</strong>,
});
```

(simplified example, see [full code](https://github.com/AdrianMrn/tado-temperature-notifier/blob/master/src/notifier.ts))

That wraps it up!
TODO: screenshot of incoming email

## Bonus round: optimization through caching

I managed to improve the script run time from 7 seconds down to 2 seconds by caching some values in the global scope.

Since Google Cloud Functions will try to ["recycle the execution environment of a previous invocation"](https://cloud.google.com/functions/docs/bestpractices/tips#use_global_variables_to_reuse_objects_in_future_invocations), any variables that are instantiated in the global scope of your script have a large possibility of still being available the next time you run your script.

This means that for remote values (API, DB …) that you don't expect to change without your interference, you don't need to retrieve them every time the script runs, but only when the variable is empty.

I used this trick to cache my Tado home details (not the temperature), as these shouldn't change in between executions of the script:

```js
let cachedHomeZones = {};
export function getHomeZones(tadoToken, homeId) {
  return new Promise(async (resolve) => {
    if (cachedHomeZones[homeId]) {
      return resolve(cachedHomeZones[homeId]);
    }

    const endpoint = `https://my.tado.com/api/v2/homes/${homeId}/zones`;
    const response = await Axios.get(endpoint, {
      headers: { Authorization: `Bearer ${tadoToken}` },
    });

    cachedHomeZones[homeId] = response.data;
    resolve(response.data);
  });
}
```

(simplified example, see [full code](https://github.com/AdrianMrn/tado-temperature-notifier/blob/master/src/tado.ts))

The values in the Google Cloud Datastore table also shouldn't change between two runs, since only the script itself can make changes to this table. This means the database only needs to be queried whenever there are no cached values available:

```js
let cachedOldShouldOpenWindows, cachedOldShouldCloseCurtains;
export function getOldValues() {
  return new Promise(async (resolve) => {
    if (
      cachedOldShouldOpenWindows !== undefined &&
      cachedOldShouldCloseCurtains !== undefined
    ) {
      return resolve({
        cachedOldShouldOpenWindows,
        cachedOldShouldCloseCurtains,
      });
    }

    const response = await datastore.get(datastore.key(["state", "oldValues"]));
    const { oldShouldOpenWindows, oldShouldCloseCurtains } = response[0];

    cachedOldShouldOpenWindows = oldShouldOpenWindows;
    cachedOldShouldCloseCurtains = oldShouldCloseCurtains;

    resolve({ oldShouldOpenWindows, oldShouldCloseCurtains });
  });
}
```

(simplified example, see [full code](https://github.com/AdrianMrn/tado-temperature-notifier/blob/master/src/datastore.ts))

Keep in mind that a warm boot can never be guaranteed, so you shouldn't try to persist any data in the global scope without also properly storing it elsewhere.

This trick saves the script 2 API calls and 1 DB read each time the script runs from a recycled execution environment, which results in 5 seconds saved each time the function runs.

It can also save you money. If you have a script that has to run very often, its monthly cost can be lowered considerably by decreasing the amount of Datastore calls and internet egress (outbound API calls).

This also works in Node.js scripts run on AWS Lambda Node, as they also use warm boots.

## Cost and summary

The script takes around 2-7 seconds to run and uses a maximum of 128 MB of RAM. It gets invocated by Cloud Scheduler about 9,000 times per month, and it sends out around 100 emails per month.

Considering all these metrics, my script should stay well withing both GCP's and SendGrid's free tiers, bringing the total monthly recurring cost of this project to \$0.

GCP has been really fun and easy to work with, especially compared to AWS, in my opinion. I'll definitely consider using it again for any similar problems I encounter.

After finishing the project and having it running for a couple of days, I realised I don't check my emails all that often, and the windows didn't really get opened and closed on schedule. The highest temperature I measured inside of the house was a very uncomfortable 30.4° C.

My girlfriend and I have decided that we're buying a portable air conditioner next summer. Any recommendations are welcome @AdriaanMrn!

## Resources

- https://shkspr.mobi/blog/2019/02/tado-api-guide-updated-for-2019/
- https://cloud.google.com/docs
- https://github.com/sendgrid/sendgrid-nodejs
