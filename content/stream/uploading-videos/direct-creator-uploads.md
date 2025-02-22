---
pcx-content-type: tutorial
title: Direct creator uploads
weight: 4
---

# Direct creator uploads

Direct creator uploads allow users to upload videos without API tokens. A common place to
use Direct creator uploads is on web apps, client side applications, or on mobile apps
where users upload content directly to Stream.

{{<Aside>}}

Pending uploads count towards your storage limit.

{{</Aside>}}

## Generate a unique one-time upload URL

To give users the ability to directly upload their videos, first generate and
provide them with a unique one-time upload URL with the following API request.

To make API requests you will need your [Cloudflare API token](https://www.cloudflare.com/a/account/my-account) and your Cloudflare [account ID](https://www.cloudflare.com/a/overview/).

### Upload constraints

There are several constraints you can enforce on your user's uploads through the
body of the `POST` request:

{{<definitions>}}

- `maxDurationSeconds` {{<type>}}integer{{</type>}} {{<prop-meta>}}required{{</prop-meta>}}

  - Enforces the maximum duration in seconds for a video the user uploads. For direct uploads, Stream requires videos are at least 1 second in length, and restricts to a maximum of 6 hours. Therefore, this field must be greater than 1 and less than 21,600.

- `expiry` {{<type>}}string (date){{</type>}} {{<prop-meta>}}default: now + 30 minutes{{</prop-meta>}}
  - Optional string field that enforces the time after which the unique one-time upload URL is invalid. The time value must be formatted in RFC3339 layout and will be interpreted against UTC time zone. If an expiry is set, it must be no less than two minutes in the future, and not more than 6 hours in the future. If an expiry is not set, the upload URL will expire 30 minutes after it's creation.

{{</definitions>}}

Additionally, you can control security features through these fields:

{{<definitions>}}

- `requireSignedURLs` {{<type>}}boolean{{</type>}} {{<prop-meta>}}default: false{{</prop-meta>}}

  - Limits the permission to view the video to only [signed URLs](/stream/viewing-videos/securing-your-stream/).

- `allowedOrigins` {{<type>}}array of strings{{</type>}} {{<prop-meta>}}default: _empty_{{</prop-meta>}}

  - Limit the domains this video can be embedded on. Learn more about [allowed origins](/stream/viewing-videos/securing-your-stream/).

- `thumbnailTimestampPct` {{<type>}}float{{</type>}} {{<prop-meta>}}default: 0{{</prop-meta>}}

  - Sets the timestamp location of [thumbnail](/stream/viewing-videos/displaying-thumbnails/) image to a percentage location of the video from 0 to 1.

- `watermark` {{<type>}}string{{</type>}} {{<prop-meta>}}default: _none_{{</prop-meta>}}

  - `uid` of the watermark profile to be included in this video. Video uploaded by the link will be [watermarks](/stream/uploading-videos/applying-watermarks/) automatically.

- `meta` {{<type>}}json map{{</type>}} {{<prop-meta>}}default: _none_{{</prop-meta>}}
  - Set the video's `name` along with any other additional arbitrary keys for metadata to be stored.

{{</definitions>}}

## Using Basic Uploads

If the uploads from your creators are under 200MB, you can use basic uploads. For videos over 200 MB, use TUS uploads (described later in this article.)

The first step is to request a token by calling the `direct_upload` end point from your server:

```bash
curl -X POST \
 -H 'Authorization: Bearer $TOKEN' \
https://api.cloudflare.com/client/v4/accounts/$ACCOUNT/stream/direct_upload \
 --data '{
    "maxDurationSeconds": 3600,
    "expiry": "2020-04-06T02:20:00Z",
    "requireSignedURLs": true,
    "allowedOrigins": ["example.com"],
    "thumbnailTimestampPct": 0.568427,
    "watermark": {
        "uid": "$watermark_uid"
    }
 }'
```

### Example responses

A successful response looks like:

```bash
{
  "result": {
    "uploadURL": "https://upload.videodelivery.net/f65014bc6ff5419ea86e7972a047ba22",
    "uid": "f65014bc6ff5419ea86e7972a047ba22"
  },
  "success": true,
  "errors": [],
  "messages": []
}
```

An unsuccessful response might look like:

```bash
{
  "result": null,
  "success": false,
  "errors": [
    {
      "code": 10005,
      "message": "Bad Request"
    }
  ],
  "messages": [
    {
      "code": 10005,
      "message": "required field maxDurationSeconds is missing"
    }
  ]
}
```

The `uploadURL` provided in the `result` body of a successful request should be passed along to the end-user to make their upload request.

The `uid` references the reserved media object's unique identifier and can be kept as a reference to query our [API](/stream/uploading-videos/searching/).

## Direct creator upload request from end users

Using the `uploadURL` provided in the previous request, users can upload video
files. Uploads are limited to 200 MB in size.

```bash
curl -X POST \
  -F file=@/Users/mickie/Downloads/example_video.mp4 \
  https://upload.videodelivery.net/f65014bc6ff5419ea86e7972a047ba22
```

A successful upload will receive a `200` response. If the upload does not meet
the upload constraints defined at time of creation or is larger than 200 MB in
size, the user will receive a `4xx` response.

### Example

```html
<!DOCTYPE html>
<html lang="en">
  <head></head>
  <body>
    <form id="form">
      <input type="file" accept="video/*" id="video" />
      <button type="submit">Upload Video</button>
    </form>
    <script>
      async function getOneTimeUploadUrl() {
        // The real implementation of this function should make an API call to your server
        // where a unique one-time upload URL should be generated and returned to the browser.
        // Here we will use a fake one that looks real but won't actually work.
        return 'https://upload.videodelivery.net/f65014bc6ff5419ea86e7972a047ba22';
      }

      const form = document.getElementById('form');
      const videoInput = document.getElementById('video');

      form.addEventListener('submit', async e => {
        e.preventDefault();
        const oneTimeUploadUrl = await getOneTimeUploadUrl();
        const video = videoInput.files[0];
        const formData = new FormData();
        formData.append('file', video);
        const uploadResult = await fetch(oneTimeUploadUrl, {
          method: 'POST',
          body: formData,
        });
        form.innerHTML = '<h3>Upload successful!</h3>';
      });
    </script>
  </body>
</html>
```

## Using tus (recommended for videos over 200MB)

tus is a protocol that supports resumable uploads and works best for larger files.

Typically, tus uploads require the authentication information to be sent with every request. This is not ideal for direct creators uploads because it exposes your API key (or token) to the end user.

To get around this, you can request a one-time tokenized URL by making a POST request to the `/stream?direct_user=true` end point:

```bash
curl -H "Authorization: bearer $TOKEN" -X POST -H 'Tus-Resumable: 1.0.0' -H 'Upload-Length: $VIDEO_LENGTH' 'https://api.cloudflare.com/client/v4/accounts/$ACCOUNT/stream?direct_user=true'
```

The response will contain a `Location` header which provides the one-time URL the client can use to upload the video using tus.

Here is a demo Cloudflare Worker script which returns the one-time upload URL:

```js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

/**
 * Respond to the request
 * @param {Request} request
 */
async function handleRequest(request) {
  const response = await fetch(
    'https://api.cloudflare.com/client/v4/accounts/$ACCOUNT/stream?direct_user=true',
    {
      method: 'POST',
      headers: {
        'Authorization': 'bearer $TOKEN',
        'Tus-Resumable': '1.0.0',
        'Upload-Length': request.headers.get('Upload-Length'),
        'Upload-Metadata': request.headers.get('Upload-Metadata'),
      },
    }
  );

  const destination = response.headers.get('Location');

  return new Response(null, {
    headers: {
      'Access-Control-Expose-Headers': 'Location',
      'Access-Control-Allow-Headers': '*',
      'Access-Control-Allow-Origin': '*',
      'Location': destination,
    },
  });
}
```

Once you have an endpoint that returns the tokenized upload URL from the `location` header, you can use it by setting the tus client to make a request to _your_ endpoint. For details on using a tus client, refer to the [Resumable uploads with tus ](/stream/uploading-videos/upload-video-file/#resumable-uploads-with-tus-for-large-files) article.

### Testing your Direct Creator Upload Endpoint

Once you have built your endpoint which calls Stream and returns the tokenized URL in the location header, you can test it with this [tus codepen demo](https://codepen.io/cfzf/pen/wvGMRXe). In the demo codepen, paste your end point URL in the `Upload endpoint` field and then try to upload a video.

When using Direct Creator Uploads, the `Upload endpoint` field in the demo should contain the url to your endpoint, not to the videodelivery.net tokenized URL. This is the most common reason Direct Creator Uploads fail using tus. Customers often set the tus url to the videodelivery.net URL instead of to their endpoint which _returns_ the videodelivery.net URL.

Please note that if you are developing on localhost, your test using the codepen may fail. Before testing, it is best to push your endpoint to a server with an IP and/or domain so you are not using localhost. Alternatively, you can setup a Worker with the example code provided above.

### Upload-Metadata header syntax

You can apply the same constraints as Direct Creator Upload via basic upload when using tus. To do so, you must pass the expiry and maxDurationSeconds as part of the `Upload-Metadata` request header as part of the first request (made by the Worker in the example above.) The `Upload-Metadata` values are ignored from subsequent requests that do the actual file upload.

Upload-Metadata header should contain key-value pairs. The keys are text and the values should be base64. Separate the key and values by a space, _not_ an equal sign. To join multiple key-value pairs, include a comma with no additional spaces.

In the example below, the `Upload-Metadata` header is instructing Stream to only accept uploads with max video duration of 10 minutes and to make this video private:

`'Upload-Metadata: maxDurationSeconds NjAw,requiresignedurls'`

_NjAw_ is the base64 encoded value for "600" (or 10 minutes).

## Tracking user upload progress

After the creation of a unique one-time upload URL, you may wish to retain the
`uid` returned in the response to track the progress of a user's upload.

You can do that two ways:

1.  You can [query the media API](/stream/uploading-videos/searching/) with the UID
    to understand it's status.

2.  You can [create a webhook subscription](/stream/uploading-videos/using-webhooks/) to receive notifications
    regarding the status of videos. These notifications include the video's UID.
