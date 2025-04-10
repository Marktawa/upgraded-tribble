# How to Set Up a Newsletter Subscription Service with Strapi, Brevo, and Next.js

## Introduction

## Prerequisites

- [Node.js LTS version 18, 20, or 22](https://nodejs.org/en/download)
- [Brevo account](https://onboarding.brevo.com/account/register)

## Set up Strapi

Open your work directory in your terminal and create a Strapi app named `newsletter` using the following command:
```shell
npx create-strapi-app@latest newsletter
```

Answer the prompts as follows:
```
Ok to proceed? (y) y
? Please log in or sign up. Skip
? Do you want to use the default database (sqlite) ? Yes
? Start with an example structure & data? No
? Start with Typescript? No
? Install dependencies with npm? Yes
? Initialize a git repository? No
```

Create admin user for your Strapi app inside your Strapi app folder, `newsletter`:
```shell
cd newsletter
```

```shell
npm run strapi admin:create-user -- --firstname=Kai --lastname=Doe --email=chef@strapi.io --password=Gourmet1234
```

## Send email

### Generate API

Create an api and name it `email-news`. You will use this to send emails from the Strapi backend server.
```shell
npm run strapi generate
```

Choose `api - Generate a basic API`. Name it `email-news`. Choose `no` for `is this api for a plugin?`.

The following files will be generated:
- /api/email-news/routes/email-news.js
- /api/email-news/controllers/email-news.js
- /api/email-news/services/email-news.js

### Configure Brevo

<!--How do you send an email using Strapi and Brevo API
- Get API key
- Add API key to env var -->

[Sign in to your Brevo account](https://login.brevo.com) and generate a new API key in the [API keys page](https://app.brevo.com/settings/keys/api).

![Brevo API key dashboard with button to generate new API key](https://res.cloudinary.com/craigsims808/image/upload/v1740972567/strapi/newsletter/generate-brevo-api-key_mk73dz.png)

Add your Brevo API key to your environment variables using the `.env` file.

```
BREVO_API_KEY=Your-newly-generated-Brevo-API-key
```

Install the Brevo SDK.
```shell
npm install sib-api-v3-sdk
```

### Create custom email service

Create a custom email service `newsletter/src/api/email-news/services/email-news.js`
```js
"use strict";

const SibApiV3Sdk = require("sib-api-v3-sdk");

module.exports = {
  async sendEmail({ to, subject, htmlContent }) {
    try {
      // Initialize Brevo API client
      const defaultClient = SibApiV3Sdk.ApiClient.instance;
      const apiKey = defaultClient.authentications["api-key"];
      apiKey.apiKey = process.env.BREVO_API_KEY;

      // Configure the email sender and recipient
      const apiInstance = new SibApiV3Sdk.TransactionalEmailsApi();
      const sendSmtpEmail = new SibApiV3Sdk.SendSmtpEmail();

      sendSmtpEmail.sender = { name: "Your Business", email: "your-email@example.com" };
      sendSmtpEmail.to = [{ email: to }];
      sendSmtpEmail.subject = subject;
      sendSmtpEmail.htmlContent = htmlContent;

      // Send the email
      const response = await apiInstance.sendTransacEmail(sendSmtpEmail);
      return response;
    } catch (error) {
      console.error("Error sending email:", error);
      throw new Error("Failed to send email");
    }
  },
};
```

This will initialize the Brevo API client, configure the email sender and recipient and then send the email.

### Create API controller

Create an API controller `send` in `newsletter/src/api/email-news/controllers/email-news.js`.
```js
"use strict";

module.exports = {
  async send(ctx) {
    try {

      const { to, subject, htmlContent } = ctx.request.body;

      if (!to || !subject || !htmlContent) {
        return ctx.badRequest("Missing required fields: to, subject, htmlContent");
      }

      // Access service using `strapi.service()`
      await strapi.service("api::email-news.email-news").sendEmail({ to, subject, htmlContent });

      ctx.send({ message: "Email sent successfully" });
    } catch (error) {
      console.error("Email error:", error);
      ctx.send({ error: "Failed to send email", details: error.message });
    }
  },
};
```

This controller checks for required fields for the email then uses the `sendEmail` service we created earlier to send the email.

### Create API routes

Create API Routes
```js
module.exports = {
  routes: [
    {
      method: "POST",
      path: "/send-email",
      handler: "email-news.send",
      config: {
        auth: false, // Set to true if authentication is required
      },
    },
  ],
};
```

We have added a route `/send-email` for the `send` controller.

### Test email API

Run Strapi server.
```shell
npm run develop
```

Run the following command in a new terminal session:
```shell
curl -X POST http://localhost:1337/api/send-email \
     -H "Content-Type: application/json" \
     -d '{
           "to": "munaluladev@gmail.com",
           "subject": "Test 1: Welcome!",
           "htmlContent": "<p>Thank you for signing up!</p>"
         }'
```

>**NOTE:** *Make sure to replace the email with one whose inbox you have access to.*

Verify email delivery, you will get this response:
```json
{ "message": "Email sent successfully" }
```

Check your mailbox to confirm delivery.
![Email inbox showing delivered email 1](https://res.cloudinary.com/craigsims808/image/upload/v1741222353/strapi/newsletter/check-mailbox-2_najp6p.png)

## Send email to collection (1 user)

Now that we know that our Strapi email API is working and can send emails via Brevo API we can test sending emails to a collection created in the Strapi admin dashboard.

### Create Subscriber collection

[Visit your Strapi Admin dashboard in your browser](http://localhost:1337/admin) and create a new collection **Subscriber**. Give it two fields: 
- A text field called **name**.
- An email field called **email**.

![Subscriber collection with two fields: name and email](https://res.cloudinary.com/craigsims808/image/upload/v1740974341/strapi/newsletter/subscriber-collection_vof5lq.png)

Add an entry to your collection using the **Content Manager** in the Strapi Admin Dashboard. Use an email address whose inbox you can access.

![Subscriber collection with one entry](https://res.cloudinary.com/craigsims808/image/upload/v1741222840/strapi/newsletter/subscriber-collection-single-entry_bkdlp2.png)

Enable public read access(`find` and `findOne`) to the Subscriber collection.

Click **Settings** then **Users & Permissions Plugin** then **Roles** and then **Public**.

![Subscriber collection with Public Read Access](https://res.cloudinary.com/craigsims808/image/upload/v1741223825/strapi/newsletter/enable-public-access-subscriber-collection_l42azy.png)

### Add new API controller

The objective is to fetch an entry from the Subscriber collection using the `documentId` of the entry. Use `email` and `name` fields to construct and send an email to the user. <!--This method uses the [Entity Service API]().-->

Add a new controller, `sendToSubscriber` in `newsletter/src/api/email-news/controllers/email-news.js` below the `send` controller we created earlier.
```js
"use strict";

module.exports = {
  //... Previous code

  async sendToSubscriber(ctx) {
    try {
      const { id, subject, htmlContent } = ctx.request.body;
      // id in this case is documentId

      if (!id || !subject || !htmlContent) {
        return ctx.badRequest("Missing required fields: id, subject, htmlContent");
      }

      // Fetch subscriber from database

      //Using Entity Service API
      /*const subscriber = await strapi.entityService.findOne("api::subscriber.subscriber", id, {
        fields: ["name", "email"],
      });
      */

      //Using Document Service API 
      const subscriber = await strapi.documents("api::subscriber.subscriber").findOne({
        documentId: id,
        fields: ["name", "email"],        
      });

      if (!subscriber) {
        return ctx.notFound("Subscriber not found");
      }

      const { email, name } = subscriber;

      // Send email
      await strapi.service("api::email-news.email-news").sendEmail({
        to: email,
        subject: subject.replace("{name}", name),
        htmlContent: htmlContent.replace("{name}", name), // Replace {name} with subscriber's name
      });

      ctx.send({ message: `Email sent successfully to ${email}` });
    } catch (error) {
      console.error("Email error:", error);
      ctx.send({ error: "Failed to send email", details: error.message });
    }
  },
};
```

The `sendToSubscriber` controller fetches the subscriber entry from the collection using a `documentId`, retrieves the corresponding `name` and `email` then sends an email using the `sendEmail` service.

### Add new API Route

Update `newsletter/src/api/email-news/routes/email-news.js`.
```js
module.exports = {
  routes: [
    {
      //... Previous Code
      method: "POST",
      path: "/send-email-to-subscriber",
      handler: "email-news.sendToSubscriber",
      config: {
        auth: false, // Set to true if authentication is required
      },
    },
  ],
};
```

We have added a new route `/send-email-to-subscriber` for the `sendToSubscriber` controller.

### Test API

<!--
Restart Strapi server and then test the API:
```shell
curl -X POST http://localhost:1337/api/send-email-to-subscriber \
     -H "Content-Type: application/json" \
     -d '{
            "id": 1,
            "subject": "Test 2: Hello, {name}!",
            "htmlContent": "<p>Dear {name}, thank you for subscribing!</p>"
         }'
```
-->

<!-- Document Service API alternative -->
First retrieve the `documentId` for the Subscriber entry you created earlier.

```shell
curl http://localhost:1337/api/subscribers?fields%5B0%5D=documentId
```

>**NOTE:**
>
>*This query uses field selection to know more about this check out [Population and Filtering: Field Selection](https://docs.strapi.io/dev-docs/api/rest/populate-select#field-selection)* in the Strapi docs

The result will look something like this:

```json
{
    "data": [
        {
            "id": 3,
            "documentId": "ud8fkwy24ga5xm7iduc5wjyk"
        }
    ],
}
```

Copy the `documentId` and use it to test the API route, `localhost:1337/api/send-email-to-subscriber` using the following command:

```shell
curl -X POST http://localhost:1337/api/send-email-to-subscriber \
     -H "Content-Type: application/json" \
     -d '{
            "id": "ud8fkwy24ga5xm7iduc5wjyk",
            "subject": "Test 2: Hello, {name}!",
            "htmlContent": "<p>Dear {name}, thank you for subscribing!</p>"
         }'
```

Check your mailbox to confirm receipt of the email.

![Email inbox showing delivered email 2](https://res.cloudinary.com/craigsims808/image/upload/v1741224541/strapi/newsletter/check-mailbox-single-subscriber_st2teb.png)

## Send email to collection (All users)

### Add Multiple Entries to Subscriber collection

Add a new entry to your collection using the **Content Manager** in the Strapi Admin Dashboard. Use an email address whose inbox you can access.

Your **Subscriber** collection should now have more than one entry.

![Subscriber collection with multiple entries](https://res.cloudinary.com/craigsims808/image/upload/v1741229710/strapi/newsletter/subscriber-collection-multiple-entries_rfmkqa.png)

### Add New API Controller

Add a new controller, `sendToAllSubscribers` in `newsletter/src/api/email-news/controllers/email-news.js` below the `sendToSubscriber` controller.

```js
"use strict";

module.exports = {
  //... Previous code

  async sendToAllSubscribers(ctx) {
    try {
      const { subject, htmlContent } = ctx.request.body;
  
      if (!subject || !htmlContent) {
        return ctx.badRequest("Missing required fields: subject, htmlContent");
      }
  
      // Fetch all subscribers

      //Using Entity Service API
      /*
      const subscribers = await strapi.entityService.findMany("api::subscriber.subscriber", {
        fields: ["name", "email"],
      });
      */

     //Using Document Service API
     const subscribers = await strapi.documents("api::subscriber.subscriber").findMany({
        fields: ["name", "email"],
     });
  
      if (subscribers.length === 0) {
        return ctx.notFound("No subscribers found");
      }
  
      // Send emails to all subscribers
      for (const subscriber of subscribers) {
        await strapi.service("api::email-news.email-news").sendEmail({
          to: subscriber.email,
          subject: subject.replace("{name}", subscriber.name),
          htmlContent: htmlContent.replace("{name}", subscriber.name),
        });
      }
  
      ctx.send({ message: `Emails sent to ${subscribers.length} subscribers` });
    } catch (error) {
      console.error("Email error:", error);
      ctx.send({ error: "Failed to send emails", details: error.message });
    }
  }  
};
```

The `sendToAllSubscribers` controller fetches all the subscriber entries from the **Subscriber** collection, retrieves the corresponding `name` and `email` for each **Subscriber** then sends an email using the `sendEmail` service.

### Add New API Route

Add a new route `/send-email-to-all` for the `sendToAllSubscribers` controller in `newsletter/src/api/email-news/routes/email-news.js`.
```js
module.exports = {
  routes: [
      //... Previous Code
    {
      method: "POST",
      path: "/send-email-to-all",
      handler: "email-news.sendToAllSubscribers",
      config: { auth: false },
    }
  ],
};
```

### Test API

Restart Strapi server then test the API:

```shell
curl -X POST http://localhost:1337/api/send-email-to-all \
     -H "Content-Type: application/json" \
     -d '{
           "subject": "Test 3: All Hello, {name}!",
           "htmlContent": "<p>Dear {name}, thank you for subscribing!</p>"
         }'
```

Check the mailboxes of emails in the **Subscriber** collection to confirm receipt of the email.

![Email inbox showing delivered email 3](https://res.cloudinary.com/craigsims808/image/upload/v1741230681/strapi/newsletter/check-mailbox-all-subscribers_pdjsie.png)

## Send Newsletter collection to Subscribers collection

### Create Newsletter collection

Open your Strapi Admin and create a new collection **Newsletter**. Give it two fields: 
- A text field(Short text) called **title**.
- A text field(Long text) called **content**.

![Newsletter collection creation](https://res.cloudinary.com/craigsims808/image/upload/v1741321043/strapi/newsletter/newsletter-collection-creation_pagr3s.png)

Add an entry to your Newsletter collection using the **Content Manager**.

![Newsletter collection entry](https://res.cloudinary.com/craigsims808/image/upload/v1741321043/strapi/newsletter/newsletter-collection-creation_pagr3s.png)

Enable public read access (`find` and `findOne`) to the **Newsletter** collection.

Click **Settings** then **Users & Permissions Plugin** then **Roles** and then **Public**.

![Newsletter enable read access](https://res.cloudinary.com/craigsims808/image/upload/v1741321043/strapi/newsletter/newsletter-collection-public-access_nvver4.png)

### Create a Controller to Send Newsletters

Create a new controller `sendNewsletter` inside `src/api/newsletter/controllers/newsletter.js`:
```js
"use strict";

const { createCoreController } = require('@strapi/strapi').factories;

module.exports = createCoreController('api::newsletter.newsletter', ({ strapi }) => ({
  async sendNewsletter(ctx) {
    try {
      const { id } = ctx.request.body;
      // id in this case is documentId

      if (!id) {
        return ctx.badRequest("Missing required field: id");
      }

      // Fetch the newsletter from the collection

      // Using Entity Service API
      /*const newsletter = await strapi.entityService.findOne("api::newsletter.newsletter", id, {
        fields: ["title", "content"],
      });
      */

      // Using Document Service API
      const newsletter = await strapi.documents("api::newsletter.newsletter").findOne({
        documentId: id,
        fields: ["title", "content"],
      });

      if (!newsletter) {
        return ctx.notFound("Newsletter not found");
      }

      const { title, content } = newsletter;

      // Fetch all subscribers

      //Using Entity Service API
      /*
      const subscribers = await strapi.entityService.findMany("api::subscriber.subscriber", {
        fields: ["name", "email"],
      });
      */

      //Using Document Service API
      const subscribers = await strapi.documents("api::subscriber.subscriber").findMany({
        fields: ["name", "email"],
      });

      if (!subscribers || subscribers.length === 0) {
        return ctx.notFound("No subscribers found");
      }

      // Send the newsletter to all subscribers
      for (const subscriber of subscribers) {
        const personalizedContent = content.replace("{name}", subscriber.name);

        await strapi.service("api::email-news.email-news").sendEmail({
          to: subscriber.email,
          subject: title, // Use the newsletter title as the email subject
          htmlContent: `<p>Dear ${subscriber.name},</p><p>${personalizedContent}</p>`,
        });
      }

      ctx.send({ message: `Newsletter sent to ${subscribers.length} subscribers` });
    } catch (error) {
      console.error("Email error:", error);
      ctx.send({ error: "Failed to send newsletter", details: error.message });
    }
  },
}));
```

<!--
The `sendToAllSubscribers` controller fetches all the subscriber entries from the **Subscriber** collection, retrieves the corresponding `name` and `email` for each **Subscriber** then sends an email using the `sendEmail` service.
-->

This `sendNewsletter` controller fetches a newsletter entry from the **Newsletter** collection, fetches all the subscribers from the **Subscriber** collection then sends an email containing the newsletter to using the `sendEmail` service to each subscriber.

### Create a Custom Route to Send Newsletters

Create a `send-newsletter` route to be handled by the `sendNewsletter` controller in a new file `src/api/newsletter/routes/custom-routes.js`:

```js
module.exports = {
  routes: [
    {
      method: "POST",
      path: "/send-newsletter",
      handler: "newsletter.sendNewsletter",
      config: {
        auth: false, // Set to true if authentication is required
      },
    },
  ],
};
```

### Test API

<!--
```shell
curl -X POST http://localhost:1337/api/send-newsletter \
     -H "Content-Type: application/json" \
     -d '{
           "id": 1
         }'

```
-->

<!-- Document Service API alternative -->
First retrieve the `documentId` for the Newsletter entry you created earlier.

```shell
curl http://localhost:1337/api/newsletters?fields%5B0%5D=documentId
```

The result will look something like this:

```json
{
    "data": [
        {
            "id": 1,
            "documentId": "rxaoulexwvykziyuz25bqi8y"
        }
    ],
}
```

Copy the `documentId` and use it to test the API route, `localhost:1337/api/send-newsletter` using the following command:

```shell
curl -X POST http://localhost:1337/api/send-newsletter \
     -H "Content-Type: application/json" \
     -d '{
            "id": "rxaoulexwvykziyuz25bqi8y"
         }'
```

Check your mailbox.
![check mailbox 4](https://res.cloudinary.com/craigsims808/image/upload/v1741326525/strapi/newsletter/check-mailbox-newsletter-1_femwg8.png)

## Automatically send Newsletter when published

This step involves sending a newsletter to subscribers as soon as it is published by making use of lifecycle hooks.

### Create Lifecycle Hook

Create a new file for lifecycle hooks in your Newsletter model:

Create `src/api/newsletter/content-types/newsletter/lifecycles.js`:

```js
// src/api/newsletter/content-types/newsletter/lifecycles.js
module.exports = {
    async afterCreate(event) {
      const { result } = event;
      //console.log(result);
      if (result.publishedAt) {
        //await sendNewsletterToSubscribers(result.id); // Entity Service API
        await sendNewsletterToSubscribers(result.documentId);
      }
    },
    
    async afterUpdate(event) {
      const { result, params } = event;
      // Check if the newsletter was just published
      if (result.publishedAt && (!params.data.publishedAt || params.data.publishedAt === result.publishedAt)) {
        //await sendNewsletterToSubscribers(result.id); // Entity Service API
        await sendNewsletterToSubscribers(result.documentId);
      }
    },
  };
  
  async function sendNewsletterToSubscribers(id) {
    try {
      // Fetch the newsletter from the Newsletter collection

      //Entity Service API
      /*
      const newsletter = await strapi.entityService.findOne("api::newsletter.newsletter", id, {
        fields: ["title", "content"],
      });
      */

      const newsletter = await strapi.documents("api::newsletter.newsletter").findOne({
        documentId: id,
        fields: ["title", "content"],
      });
  
      if (!newsletter) {
        console.error("Newsletter not found");
        return;
      }
  
      const { title, content } = newsletter;
  
      // Fetch all subscribers

      // Entity Service API
      /*
      const subscribers = await strapi.entityService.findMany("api::subscriber.subscriber", {
        fields: ["name", "email"],
      });
      */

     // Document Service API
      const subscribers = await strapi.documents("api::subscriber.subscriber").findMany({
        fields: ["name", "email"],
      });
  
      if (!subscribers || subscribers.length === 0) {
        console.log("No subscribers found");
        return;
      }
  
      // Send the newsletter to all subscribers
      for (const subscriber of subscribers) {
        const personalizedContent = content.replace("{name}", subscriber.name);
  
        await strapi.service("api::email-news.email-news").sendEmail({
          to: subscriber.email,
          subject: `Test 5: ${title}`,
          htmlContent: `<p>Dear ${subscriber.name},</p><p>${personalizedContent}</p>`,
        });
      }
  
      console.log(`Newsletter sent to ${subscribers.length} subscribers`);
    } catch (error) {
      console.error("Email error:", error);
    }
  }
```

When triggered, this lifecycle hook fetches a newsletter from the **Newsletter** collection, fetches all the subscribers from the **Subscriber** collection then sends an email containing the newsletter entry to all the subscribers using the `sendEmail` service.

The lifecycle hooks will trigger either when:

- A new newsletter is created with published status
- An existing newsletter is updated to published status

### Test

Publish a new newsletter in your Strapi Admin.

![Newly published newsletter](https://res.cloudinary.com/craigsims808/image/upload/v1741377647/strapi/newsletter/newsletter-collection-another-entry_mqver0.png)

Check the mailboxes to confirm if the newsletter was delivered to the subscriber mailing list.

![Check mailbox Newsletter automatic](https://res.cloudinary.com/craigsims808/image/upload/v1741377272/strapi/newsletter/check-mailbox-newsletter-2_ufsetl.png)

## Update **Subscribers** collection using REST API

The next phase of this project involves configuring the Strapi backend Subscriber collection API to allow create and read operations to be done.

### Enable Public Permissions (`create`, `findOne`, `find`)

Update the permissions to the Subscriber collection using the User & Permissions Plugin.

Click **Settings** then **Users & Permissions Plugin** then **Roles** and then **Public**.

The allowed actions for the Subscriber collection API should now be `create`, `find`, and `findOne`.

![Subscriber collection with Public Read and Create Access](https://res.cloudinary.com/craigsims808/image/upload/v1741378820/strapi/newsletter/enable-public-access-subscriber-collection-with-create_qoroq8.png)

### Test 

<!--Test creating a new Subscriber using curl.
```shell
curl -X POST http://localhost:1337/api/subscribers \
     -H "Content-Type: application/json" \
     -d '{
           "data": {
             "name": "Craig",
             "email": "craigsimakando@gmail.com"
           }
         }'
```
-->

Test creating a new Subscriber using curl.
```shell
curl -X POST http://localhost:1337/api/subscribers \
     -H "Content-Type: application/json" \
     -d '{
           "data": {
             "name": "John",
             "email": "john@doe.com"
           }
         }'
```

You should get a response similar to this:
```json
{
    "data": {
        "id": 10,
        "documentId": "or399je8658l18ivi5p3u4rz",
        "name": "John",
        "email": "john@doe.com",
        "createdAt": "2025-03-07T20:33:14.776Z",
        "updatedAt": "2025-03-07T20:33:14.776Z",
        "publishedAt": "2025-03-07T20:33:14.782Z"
    },
    "meta": {}
}
```

Verify the new list of subscribers.
```shell
curl -X GET http://localhost:1337/api/subscribers
```

<!--
Jane jane@email.com
Craig craig@email.com
John john@doe.com
-->

![New subscriber list](https://res.cloudinary.com/craigsims808/image/upload/v1741380040/strapi/newsletter/new-subscriber-list_fvuyo7.png)

## Create minimal subscribe form in (name, email) in Next.js

Next let's introduce the Next.js frontend into this project.

### Create Next project

Make directory `frontend`.
```shell
mkdir frontend && cd frontend
```

Install `react`, `react-dom`, and `next` as npm dependencies inside `frontend`.
```shell
npm install next@latest react@latest react-dom@latest
```

Open your `package.json` file and add the following npm scripts
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "^15.2.1",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  }
}
```

Create an `app` folder, then add a `layout.tsx` and `page.tsx` file.

```shell
mkdir app && touch app/layout.tsx app/page.tsx
```

Create root layout inside `app/layout.tsx`:
```tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

Create home page, `app/page.tsx`:
```tsx
export default function Page() {
  return (
    <form action="#" method="post">
      <label htmlFor="name">Name:</label>
      <input type="text" id="name" name="name" required />

      <label htmlFor="email">Email:</label>
      <input type="email" id="email" name="email" required />

      <button type="submit">Subscribe</button>
    </form>
  );
}
```

Run the development server.

```sh
npm run dev
```
Visit http://localhost:3000 to view your site.

![Next Initial frontend](https://res.cloudinary.com/craigsims808/image/upload/v1741380644/strapi/newsletter/next-initial-frontend_i8ncln.png)

## Update subscribers collection using Next form (Connect Next to Strapi)

### Update home page, `app/page.tsx`

Update home page, `app/page.tsx`:

```tsx
"use client";

import React from "react";
import { useState } from "react";

export default function Page() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [message, setMessage] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    try {
      const response = await fetch("http://localhost:1337/api/subscribers", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          data: { name, email },
        }),
      });

      if (!response.ok) {
        throw new Error("Failed to subscribe");
      }

      setMessage("Subscription successful!");
      setName("");
      setEmail("");
    } catch (error) {
      setMessage("Subscription failed. Try again.");
    }
  };

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <label htmlFor="name">Name:</label>
        <input
          type="text"
          id="name"
          name="name"
          value={name}
          onChange={(e) => setName(e.target.value)}
          required
        />

        <label htmlFor="email">Email:</label>
        <input
          type="email"
          id="email"
          name="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />

        <button type="submit">Subscribe</button>
      </form>

      {message && <p>{message}</p>}
    </div>
  );
}
```

This code defines a React component for the subscription form. It allows users to enter their name and email, then submit the form to subscribe. The form data is sent to the Strapi backend (`http://localhost:1337/api/subscribers`) via a `POST` request. If successful, a success message is displayed; otherwise, an error message is shown.

### Test form

Launch your Next development server and test the newsletter subscription form by entering a name and email. 

![Next Form subscription successful](https://res.cloudinary.com/craigsims808/image/upload/v1741383032/strapi/newsletter/next-form-subscription-successful_qfaccx.png)

Verify if the Subscriber collection has been updated in your Strapi Admin

![New Subscriber List](https://res.cloudinary.com/craigsims808/image/upload/v1741383306/strapi/newsletter/new-subscriber-list-2_txbmsu.png)

## Middleware

Create custom middleware in Strapi to handle form submission validation, logging requests, duplicate email check, and rate-limiting (spam prevention).

```shell
mkdir src/middlewares
```

### Logger

Create a new middlware file `src/middlewares/form-handler.js` inside the Strapi backend folder:

```js
// ./src/middlewares/form-handler.js
module.exports = (config, { strapi }) => {
  return async (ctx, next) => {
    if (ctx.path === '/api/subscribers' && ctx.method === 'POST') {
      strapi.log.info(`Name: ${ctx.request.body.data.name} Email: ${ctx.request.body.data.email}`);
      }
      await next();
    };
};
```

Register the middleware by updating `./config/middlewares.js` to load your middleware:

```js
module.exports = [
  'strapi::logger',
  'strapi::errors',
  'strapi::security',
  'strapi::cors',
  'strapi::poweredBy',
  'strapi::query',
  'strapi::body',
  'strapi::session',
  'strapi::favicon',
  'strapi::public',
  'global::form-handler',
];
```

Test the logger middleware by running the Strapi server.

Each time you make a form submission you should view the form submissions logged in your Strapi backend terminal. Something similar to the following:

```
Name: John Email: john@strapi.io
```

### Name Validation

Update `src/middlewares/form-handler.js` with the following code:

```js
// ./src/middlewares/form-handler.js
module.exports = (config, { strapi }) => {
  return async (ctx, next) => {
    if (ctx.path === '/api/subscribers' && ctx.method === 'POST') {
      const { name, email } = ctx.request.body.data;

      // Initialize errors array
      const errors = [];

      //Validate name
      if (!name) {
        errors.push({ field: 'name', message: 'Name is required' });
      } else if (name.length > 255) {
        errors.push({ field: 'name', message: 'Name must be 255 characters or less'})
      }

      // Display errors
      if (errors.length > 0) {
        ctx.status = 400;
        ctx.body = { errors };
        return; // Stop execution and don't proceed to next middleware
      }

      // Log valid submission
      strapi.log.info(`Valid Submission - Name: ${name} Email: ${email}`);
      }
      await next();
    };
};
```

We have updated `form-handler.js` with validation checks for the name submitted. It checks if the name is provided and limits the name to a maximum of 255 characters.

All validation errors are collected into an array. If an error occurs a `400` status is returned with detailed error messages. Invalid data is prevented from proceeding to the next middleware.

To display the error messages in your Next frontend, update `app/page.tsx` with the following code:

```tsx
"use client";

import React from "react";
import { useState } from "react";

export default function Page() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [message, setMessage] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    try {
      const response = await fetch("http://localhost:1337/api/subscribers", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          data: { name, email },
        }),
      });

      const data = await response.json();

      if (!response.ok) {
        if (data.errors) {
          // Display validation error
          setMessage(data.errors.map(err => `${err.field}: ${err.message}`).join(', '));
        } else {
          throw new Error("Failed to subscribe");
        }
      } else {
        setMessage("Subscription successful!");
        setName("");
        setEmail("");
      }
    } catch (error) {
      setMessage("Subscription failed. Try again.");
    }
  };

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <label htmlFor="name">Name:</label>
        <input
          type="text"
          id="name"
          name="name"
          value={name}
          onChange={(e) => setName(e.target.value)}
          required
        />

        <label htmlFor="email">Email:</label>
        <input
          type="email"
          id="email"
          name="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />

        <button type="submit">Subscribe</button>
      </form>

      {message && <p>{message}</p>}
    </div>
  );
}
```

This setup:
- Prevents invalid names from being saved in the **Subscriber** collection
- Provides feedback to users about validation issues
- Logs only valid submissions to the server logs

Test the name validation middleware by sending a blank name or providing a name that exceeds 255 characters.

Use browser Developer Tools (`Ctrl` + `Shift` + `I`) to make the form submissions

### Email Validation

Update `src/middlewares/form-handler.js` with the following code:

```js
// ./src/middlewares/form-handler.js
module.exports = (config, { strapi }) => {
  return async (ctx, next) => {
    if (ctx.path === '/api/subscribers' && ctx.method === 'POST') {
      const { name, email } = ctx.request.body.data;

      // Initialize errors array
      const errors = [];

      //Validate name
      if (!name) {
        errors.push({ field: 'name', message: 'Name is required' });
      } else if (name.length > 255) {
        errors.push({ field: 'name', message: 'Name must be 255 characters or less'})
      }

      // Validate email
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!email) {
        errors.push({ field: 'email', message: 'Email is required' });
      } else if (!emailRegex.test(email)) {
        errors.push({ field: 'email', message: 'Please provide a valid email address' });
      }

      // Display errors
      if (errors.length > 0) {
        ctx.status = 400;
        ctx.body = { errors };
        return; // Stop execution and don't proceed to next middleware
      }

      // Log valid submission
      strapi.log.info(`Valid Submission - Name: ${name} Email: ${email}`);
      }
      await next();
    };
};
```

This setup:
- Checks if email is provided
- Validates email format using a regex pattern `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`

Test the email validation middleware by sending a blank email or providing an invalid email.

Use browser Developer Tools (`Ctrl` + `Shift` + `I`) to make the form submissions.

### Duplicate Email Check

In addition to email validation, you can prevent duplicate email entries by checking for existing emails.

Update `src/middlewares/form-handler.js` with the following code:

```js
// ./src/middlewares/form-handler.js
module.exports = (config, { strapi }) => {
  return async (ctx, next) => {
    if (ctx.path === '/api/subscribers' && ctx.method === 'POST') {
      const { name, email } = ctx.request.body.data;

      // Initialize errors array
      const errors = [];

      //Validate name
      if (!name) {
        errors.push({ field: 'name', message: 'Name is required' });
      } else if (name.length > 255) {
        errors.push({ field: 'name', message: 'Name must be 255 characters or less'})
      }

      // Validate email
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!email) {
        errors.push({ field: 'email', message: 'Email is required' });
      } else if (!emailRegex.test(email)) {
        errors.push({ field: 'email', message: 'Please provide a valid email address' });
      }

      // Check for existing email to prevent duplicates
      if (email && emailRegex.test(email)) {
        try {
          const existingSubscriber = await strapi.documents("api::subscriber.subscriber").findMany({
            filters: { email: email },
          });

          if (existingSubscriber && existingSubscriber.length > 0) {
            errors.push({ field: 'email', message: 'This email is already subscribed' });
          }
        } catch (error) {
          strapi.log.error('Error checking for existing email:', error);
        }
      }

      // Display errors
      if (errors.length > 0) {
        ctx.status = 400;
        ctx.body = { errors };
        return; // Stop execution and don't proceed to next middleware
      }

      // Log valid submission
      strapi.log.info(`Valid Submission - Name: ${name} Email: ${email}`);
      }
      await next();
    };
};
```

Test the email duplication check by entering a duplicate email in your newsletter subscription form.

### Spam (Rate Limiting)

To prevent abuse of the subscription form, we can add rate limiting to the middleware.

Update `src/middlewares/form-handler.js` with the following code:

```js
module.exports = (config, { strapi }) => {
  const rateLimitStore = new Map();
  const RATE_LIMIT = 5; // Maximum requests
  const RATE_WINDOW = 60 * 60 * 1000; // Time window in ms (1 hour)

  const getRateLimitKey = (ctx) => {
    return ctx.request.ip;
  };

  return async (ctx, next) => {
    if (ctx.path === '/api/subscribers' && ctx.method === 'POST') {
      const key = getRateLimitKey(ctx);
      const now = Date.now();

      // Get or initialize rate limit data for this key
      if (!rateLimitStore.has(key)) {
        rateLimitStore.set(key, { count: 0, resetAt: now + RATE_WINDOW});
      }

      const limitData = rateLimitStore.get(key);

      // Reset counter if time window has passed
      if (now > limitData.resetAt) {
        limitData.count = 0;
        limitData.resetAt = now + RATE_WINDOW;
      }

      // Check if rate limit exceeded
      if (limitData.count >= RATE_LIMIT) {
        ctx.status = 429; //Too many requests
        ctx.body = {
          error: 'Rate limit exceeded',
          message: 'Too many subscription attempts. Please try again later.',
          retryAfter: Math.ceil((limitDat))
        };
        return;
      }

      // Increment counter
      limitData.count++;
      rateLimitStore.set(key, limitData);

      // Add rate limit headers
      ctx.set('X-RateLimit-Limit', RATE_LIMIt.toString());
      ctx.set('X-RateLimit-Remaining', (RATE_LIMIT - limitData.count).toString());
      ctx.set('X-RateLimit-Reset', Math.ceil(limitData.resetAt / 1000).toString());

      // Continue with validation
      const { name, email } = ctx.request.body.data;

      // Initialize errors array
      const errors = [];

      // Validate name
      if (!name) {
        errors.push({ field: 'name', message: 'Name is required' });
      } else if (name.length > 255) {
        errors.push({ field: 'name', message: 'Name must be 255 characters or less'})
      }

      // Validate email
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!email) {
        errors.push({ field: 'email', message: 'Email is required' });
      } else if (!emailRegex.test(email)) {
        errors.push({ field: 'email', message: 'Please provide a valid email address' });
      }
      
      // Check for existing email to prevent duplicates
      if (email && emailRegex.test(email)) {
        try {
          const existingSubscriber = await strapi.documents("api::subscriber.subscriber").findMany({
            filters: { email: email },
          });

          if (existingSubscriber && existingSubscriber.length > 0) {
            errors.push({ field: 'email', message: 'This email is already subscribed' });
          }
        } catch (error) {
          strapi.log.error('Error checking for existing email:', error);
        }
      }

      // Display errors
      if (errors.length > 0) {
        ctx.status = 400;
        ctx.body = { errors };
        return; // Stop execution and don't proceed to next middleware
      }

      // Log valid submission
      strapi.log.info(`Valid Submission - Name: ${name} Email: ${email}`);
      }
      await next();

    };
  };
```

The rate limiting logic is as follows:
- Requests are tracked using IP address.
- A counter is incremented each time a request is made.
- A `429` error code is return when the limit is exceeded

This implementation adds rate limiting with the following features:
- In-memory rate limiting store, `rateLimitStore` using a Map to track request counts by IP address
- Configurable rate limit settings using `RATE_LIMIT` variable to set maximum number of requests (set to 5) and `RATE_WINDOW` variable to limit the time to 1 hour.

To handle the rate limiting in your Next.js frontend, update `app/page.tsx` as follows:

```tsx
"use client";

import React from "react";
import { useState } from "react";

export default function Page() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [message, setMessage] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    try {
      const response = await fetch("http://localhost:1337/api/subscribers", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          data: { name, email },
        }),
      });

      const data = await response.json();

      if (!response.ok) {
        if (response.status === 429) {
          //Handle rate limiting
          setMessage(`${data.message} Please try again in ${Math.ceil(data.retryAfter / 60)} minutes.`);
        } else if (data.errors) {
          // Display specific validation errors
          setMessage(data.errors.map(err => `${err.field}: ${err.message}`).join(', '));
        } else {
          throw new Error("Failed to subscribe");
        }
      } else {
        setMessage("Subscription successful!");
        setName("");
        setEmail("");
      }
    } catch (error) {
      setMessage("Subscription failed. Try again.");
    }
  };

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <label htmlFor="name">Name:</label>
        <input
          type="text"
          id="name"
          name="name"
          value={name}
          onChange={(e) => setName(e.target.value)}
          required
        />

        <label htmlFor="email">Email:</label>
        <input
          type="email"
          id="email"
          name="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />

        <button type="submit">Subscribe</button>
      </form>

      {message && <p>{message}</p>}
    </div>
  );
}
```

## Landing page design

We need to incoporate a landing page design that mimics a real-life situation. A good example, can be a landing page for a short story author who needs to send newsletters to avid readers. 

The landing page will have a header with the author's name, description and a few of his works. The next section is the Newsletter subscription section which will include the form we have created previously. Finally, the footer section which will include the contact details of the author.

<!--
In a single page using HTML and CSS (Tailwind CSS), create a modern looking, mobile responsive landing page for a short story author. The landing page will have 3 sections a Header, Newsletter subscription section and a Footer. The Header will the author's name, description and links to a few of his works. The newsletter subscription section will have a form that receives the name and email address of the subscriber. The footer section will have the author's contact information and copyright.
-->

For styling we will use Tailwind CSS.

Here is the HTML for the design:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Alex Morgan - Short Story Author</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 font-sans text-gray-800">
    <!-- Header Section -->
    <header class="bg-gradient-to-r from-indigo-600 to-purple-600 text-white">
        <div class="container mx-auto px-4 py-16 md:py-24">
            <div class="flex flex-col md:flex-row items-center justify-between">
                <div class="md:w-1/2 text-center md:text-left mb-8 md:mb-0">
                    <h1 class="text-4xl md:text-5xl font-bold mb-4">Alex Morgan</h1>
                    <p class="text-lg md:text-xl opacity-90 mb-6">Crafting worlds one story at a time. Award-winning author of contemporary fiction that explores the human condition through intimate character studies.</p>
                    <div class="flex flex-wrap justify-center md:justify-start gap-3">
                        <a href="#" class="bg-white text-indigo-600 px-4 py-2 rounded-lg hover:bg-opacity-90 transition duration-300">About</a>
                        <a href="#" class="bg-transparent border border-white text-white px-4 py-2 rounded-lg hover:bg-white hover:text-indigo-600 transition duration-300">Contact</a>
                    </div>
                </div>
                <div class="md:w-1/2">
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <div class="bg-white bg-opacity-20 backdrop-filter backdrop-blur-sm p-6 rounded-lg hover:bg-opacity-30 transition duration-300">
                            <h3 class="font-semibold text-xl mb-2">Echoes of Silence</h3>
                            <p class="text-sm opacity-80">A collection of interconnected stories exploring themes of isolation in modern society.</p>
                            <a href="#" class="inline-block mt-3 text-white underline hover:no-underline">Read excerpt →</a>
                        </div>
                        <div class="bg-white bg-opacity-20 backdrop-filter backdrop-blur-sm p-6 rounded-lg hover:bg-opacity-30 transition duration-300">
                            <h3 class="font-semibold text-xl mb-2">The Last Light</h3>
                            <p class="text-sm opacity-80">Winner of the National Short Fiction Prize. A poignant tale of loss and redemption.</p>
                            <a href="#" class="inline-block mt-3 text-white underline hover:no-underline">Read excerpt →</a>
                        </div>
                        <div class="bg-white bg-opacity-20 backdrop-filter backdrop-blur-sm p-6 rounded-lg hover:bg-opacity-30 transition duration-300 md:col-span-2">
                            <h3 class="font-semibold text-xl mb-2">New Release: "Beneath the Surface"</h3>
                            <p class="text-sm opacity-80">My latest collection explores the hidden currents that pull beneath ordinary lives.</p>
                            <a href="#" class="inline-block mt-3 text-white underline hover:no-underline">Learn more →</a>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </header>

    <!-- Newsletter Section -->
    <section class="py-16 bg-white">
        <div class="container mx-auto px-4">
            <div class="max-w-2xl mx-auto text-center">
                <h2 class="text-3xl font-bold mb-6">Join My Newsletter</h2>
                <p class="text-gray-600 mb-8">Subscribe to receive new story alerts, writing insights, and exclusive content directly to your inbox.</p>
                <form class="space-y-4">
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <div>
                            <label for="name" class="sr-only">Full Name</label>
                            <input type="text" id="name" placeholder="Your Name" class="w-full px-4 py-3 rounded-lg border border-gray-300 focus:outline-none focus:ring-2 focus:ring-indigo-500">
                        </div>
                        <div>
                            <label for="email" class="sr-only">Email Address</label>
                            <input type="email" id="email" placeholder="Email Address" class="w-full px-4 py-3 rounded-lg border border-gray-300 focus:outline-none focus:ring-2 focus:ring-indigo-500">
                        </div>
                    </div>
                    <button type="submit" class="w-full md:w-auto px-6 py-3 bg-indigo-600 text-white font-medium rounded-lg hover:bg-indigo-700 transition duration-300">Subscribe Now</button>
                </form>
                <p class="text-xs text-gray-500 mt-4">I respect your privacy. Unsubscribe at any time.</p>
            </div>
        </div>
    </section>

    <!-- Footer Section -->
    <footer class="bg-gray-900 text-white py-12">
        <div class="container mx-auto px-4">
            <div class="flex flex-col md:flex-row justify-between items-center">
                <div class="text-center md:text-left mb-6 md:mb-0">
                    <h3 class="text-2xl font-bold mb-2">Alex Morgan</h3>
                    <p class="text-gray-400">author@alexmorgan.com</p>
                    <p class="text-gray-400">Literary Agency: Bright Words Inc.</p>
                </div>
                <div class="flex space-x-4">
                    <a href="#" class="h-10 w-10 rounded-full bg-gray-800 flex items-center justify-center hover:bg-indigo-600 transition duration-300">
                        <svg class="h-5 w-5" fill="currentColor" viewBox="0 0 24 24" aria-hidden="true">
                            <path fill-rule="evenodd" d="M22 12c0-5.523-4.477-10-10-10S2 6.477 2 12c0 4.991 3.657 9.128 8.438 9.878v-6.987h-2.54V12h2.54V9.797c0-2.506 1.492-3.89 3.777-3.89 1.094 0 2.238.195 2.238.195v2.46h-1.26c-1.243 0-1.63.771-1.63 1.562V12h2.773l-.443 2.89h-2.33v6.988C18.343 21.128 22 16.991 22 12z" clip-rule="evenodd" />
                        </svg>
                    </a>
                    <a href="#" class="h-10 w-10 rounded-full bg-gray-800 flex items-center justify-center hover:bg-indigo-600 transition duration-300">
                        <svg class="h-5 w-5" fill="currentColor" viewBox="0 0 24 24" aria-hidden="true">
                            <path d="M8.29 20.251c7.547 0 11.675-6.253 11.675-11.675 0-.178 0-.355-.012-.53A8.348 8.348 0 0022 5.92a8.19 8.19 0 01-2.357.646 4.118 4.118 0 001.804-2.27 8.224 8.224 0 01-2.605.996 4.107 4.107 0 00-6.993 3.743 11.65 11.65 0 01-8.457-4.287 4.106 4.106 0 001.27 5.477A4.072 4.072 0 012.8 9.713v.052a4.105 4.105 0 003.292 4.022 4.095 4.095 0 01-1.853.07 4.108 4.108 0 003.834 2.85A8.233 8.233 0 012 18.407a11.616 11.616 0 006.29 1.84" />
                        </svg>
                    </a>
                    <a href="#" class="h-10 w-10 rounded-full bg-gray-800 flex items-center justify-center hover:bg-indigo-600 transition duration-300">
                        <svg class="h-5 w-5" fill="currentColor" viewBox="0 0 24 24" aria-hidden="true">
                            <path fill-rule="evenodd" d="M12.315 2c2.43 0 2.784.013 3.808.06 1.064.049 1.791.218 2.427.465a4.902 4.902 0 011.772 1.153 4.902 4.902 0 011.153 1.772c.247.636.416 1.363.465 2.427.048 1.067.06 1.407.06 4.123v.08c0 2.643-.012 2.987-.06 4.043-.049 1.064-.218 1.791-.465 2.427a4.902 4.902 0 01-1.153 1.772 4.902 4.902 0 01-1.772 1.153c-.636.247-1.363.416-2.427.465-1.067.048-1.407.06-4.123.06h-.08c-2.643 0-2.987-.012-4.043-.06-1.064-.049-1.791-.218-2.427-.465a4.902 4.902 0 01-1.772-1.153 4.902 4.902 0 01-1.153-1.772c-.247-.636-.416-1.363-.465-2.427-.047-1.024-.06-1.379-.06-3.808v-.63c0-2.43.013-2.784.06-3.808.049-1.064.218-1.791.465-2.427a4.902 4.902 0 011.153-1.772A4.902 4.902 0 015.45 2.525c.636-.247 1.363-.416 2.427-.465C8.901 2.013 9.256 2 11.685 2h.63zm-.081 1.802h-.468c-2.456 0-2.784.011-3.807.058-.975.045-1.504.207-1.857.344-.467.182-.8.398-1.15.748-.35.35-.566.683-.748 1.15-.137.353-.3.882-.344 1.857-.047 1.023-.058 1.351-.058 3.807v.468c0 2.456.011 2.784.058 3.807.045.975.207 1.504.344 1.857.182.466.399.8.748 1.15.35.35.683.566 1.15.748.353.137.882.3 1.857.344 1.054.048 1.37.058 4.041.058h.08c2.597 0 2.917-.01 3.96-.058.976-.045 1.505-.207 1.858-.344.466-.182.8-.398 1.15-.748.35-.35.566-.683.748-1.15.137-.353.3-.882.344-1.857.048-1.055.058-1.37.058-4.041v-.08c0-2.597-.01-2.917-.058-3.96-.045-.976-.207-1.505-.344-1.858a3.097 3.097 0 00-.748-1.15 3.098 3.098 0 00-1.15-.748c-.353-.137-.882-.3-1.857-.344-1.023-.047-1.351-.058-3.807-.058zM12 6.865a5.135 5.135 0 110 10.27 5.135 5.135 0 010-10.27zm0 1.802a3.333 3.333 0 100 6.666 3.333 3.333 0 000-6.666zm5.338-3.205a1.2 1.2 0 110 2.4 1.2 1.2 0 010-2.4z" clip-rule="evenodd" />
                        </svg>
                    </a>
                </div>
            </div>
            <div class="border-t border-gray-800 mt-8 pt-8 text-center">
                <p class="text-gray-400 text-sm">&copy; 2025 Alex Morgan. All rights reserved.</p>
            </div>
        </div>
    </footer>
</body>
</html>
```

## Clean Up

- Disable open access to API
- Create REST API access to subscriber collection 
- API Tokens: (Prevent read access to Newsletter, Subscriber list)
- Remove some scaffolding (Email api controller and route)
- Remove send email to collection (1 user) code - (controller and route)
- Remove send email to all subscribers code - (controller and route)
- Remove send newsletter email to all subscribers code - (controller and route)

- Does Document Service API work without the need for public access to API

## Conclusion
