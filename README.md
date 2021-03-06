# go-segmentwebhookenrich

go-segmentwebhhokenrich is a Google App Engine app used to subscribe to a [Segment.com](https://segment.com/) event webhook triggered by [Lytics](http://www.getlytics.com/) and enrich contained profile with suggested content and optimal send time, to be used for email or any other kind of interaction.

This app assumes you have a Lytics account, with at least one Segment.com trigger export running to capture segment entered and exited events. To use this app, you should configure a webhooks integration on the Segment.com source collecting these triggers. The webhook url should be the `[url of this app]/post`.

In the example code provided, we send a formatted webhook to [SparkPost](https://www.sparkpost.com/) which will deploy an email (see [strater email template](https://github.com/lytics/go-segmentwebhookenrich/blob/master/starter-template.html) for how to use the data in sparkpost) to the user at their optimal activity time including the suggested content. The base code of this app is flexible and can be edited to send the enriched data to any url.

## Configuration

There are a few configuration variables which can be set in [`config.go`](https://github.com/lytics/go-segmentwebhookenrich/blob/master/config.go)

#### 1. lyticsAPIKey `string` (required)

Your Lytics API Key is required to make content recommendation requests. It can be found by going to `Account > Manage Acccounts` while logged in to Lytics, copy the Full API key.

#### 2. webhook `string` (required)

This is a unique name of the webhook used in the app. This should match the key of the webhook details you wish to use in the `webhooks` map (see below).

#### 3. webhooks `map[string]map[string]string` (required)

This map contains the details of any possible webhook to be used in the app. The key of each should be a unique string identifier, and the value contains a map of any necessary details to be used in the POST request such as the URL, Authorization, and other headers. In the code provided, this includes the Sparkpost API Key, URL, and template ID. As is, the app will only support one out going endpoint, though it can be modified to accept different endpoints for different incoming Segment.com events.

#### 4. getOptimalHour `bool` (required)

A flag to turn on or off the inclusion of the next optimal activity time in the payload. If field is set to true we will look at the hourly data for the user and select the hour with the most activity in the past. Using this, we can include a timestamp in the payload representing the next best time to interact with this user. 

In our example, we send this as the `start_time` to the sparkpost api, meaning the email will not be sent until the next optimal hour. If this flag is set to false, or no hourly data is available for the user, the email sends immediately.


#### 5. recommendationFilter `string` (optional)

This filters down the documents returned by the Lytics content recommendation API. This is an optional configuration variable, with if not set, the recommendation API may return any web document on your website for recommendation based on the users interests. However, with this filter you can use AND/OR logic to select documents of certain urls, meta attributes, associated topics, and more to return. Consider these examples:

**URL Filter (Can use `*` as wildcard)**
>```
FILTER AND (url LIKE "www.example.com/blog/*") FROM content
```
This filter will include all documents matching the url pattern `www.example.com/blog/*` this could include `www.example.com/blog/post/1`, `www.example.com/blog/tagged/example`, etc. Be sure to choose a URL filter carefully, so as not to potentially recommend any content you wouldn't like to promote. The URL string in this filter should not contain the http:// or https:// protocol.


**Topic Filter**
>```
FILTER AND (global.Marketing > 0) FROM content
```
This filter will include all documents classified to have the topic `Marketing` with a relevence value greater than 0. You can view a list of all your topics for your content in the content section of your lytics account. All topics should be prefixed with `global.`


**Other Filters**
>```
FILTER AND (meta CONTAINS "og:type/article") FROM content
```
The filter above will only select documents with the og type article.


**Multiple Filters (AND/OR)**
>```
FILTER OR (meta CONTAINS "og:type/article", global.developers > 0) FROM content
```
```
FILTER AND (url LIKE "www.example.com/products/*", global.mobile > 0) FROM content
```

#### 6. event `*Event` (optional)

Once your webhook integration is configured your Segment source will send *all incoming events* to this app. By setting the `event` field `name` and `segment` we can select which events we actually want to process. If not set, the app will try to process all events. Currently the base code only filters one event, in the future it could be modified to recognize more.

- **event.name** `string` - The name of the segment event. Using Lytics triggers this should be `segment_entered` or `segment_exited`.
- **event.segment** `string` - The slug of the segment in Lytics (make sure API Access is enabled for the segment).


## Testing

Test with:
```sh
goapp test -v
```

If you modify the webhook, the tests should be updated as well. They will only test the base code provided, using the sparkpost example.

## Customizing & Contributing

Feel free to fork this repo and change it to suit your needs. You can change the [contents of the payload](https://github.com/lytics/go-segmentwebhookenrich/blob/master/main.go#L109) to match whatever format your endpoint expects. And [add the optimal time](https://github.com/lytics/go-segmentwebhookenrich/blob/master/main.go#L126) (returned from `evt.SendTime()` function) to the payload as you like.

If you do not use App Engine, this code can be easily adopted into another environment, the main difference would be changing the [context and client](https://github.com/lytics/go-segmentwebhookenrich/blob/master/main.go#L29).

If you find something you think could be improved in the base code you can contribute by creating a new issue, or submitting a pr for review.
