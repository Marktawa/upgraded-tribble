# How to Set Up a Newsletter Subscription Service with Strapi, Brevo, and Next.js

## Introduction

## Prerequisites
- Node.js 18 or 20
- Brevo account

## Set up Strapi

- nvm install 20

- Create Strapi
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

- Create admin
```shell
cd newsletter
```

```shell
npm run strapi admin:create-user -- --firstname=Kai --lastname=Doe --email=chef@strapi.io --password=Gourmet1234
```

## Send email

### Generate API

- Create api, name it `email-news`
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

- Install Brevo SDK
```shell
npm install sib-api-v3-sdk
```

### Create custom email service

- Create custom email service `newsletter/src/api/email-news/services/email-news.js`
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

- Create an API controller `send` in `newsletter/src/api/email-news/controllers/email-news.js`
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

### Create API routes

- Create API Routes
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

Make sure to replace the email with one whose inbox you have access to.

Verify email delivery, you will get this response:
```json
{ "message": "Email sent successfully" }
```

Check mailbox to confirm delivery
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

Enable public Read Access to the Subscriber collection.

Click Settings -> Users & Permissions Plugin -> Roles -> **Public**

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

Your Subscriber collection should now have more than one entry.

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

The `sendToAllSubscribers` controller fetches all the subscriber entries from the Subscriber collection, retrieves the corresponding `name` and `email` for each Subscriber then sends an email using the `sendEmail` service.

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

Check the mailboxes of emails in the Subscriber collection to confirm receipt of the email.

![Email inbox showing delivered email 3](https://res.cloudinary.com/craigsims808/image/upload/v1741230681/strapi/newsletter/check-mailbox-all-subscribers_pdjsie.png)

## Send Newsletter collection to subscribers collection

### Create Newsletter collection

Open your Strapi Admin and create a new collection **Newsletter**. Give it two fields: 
- A text field called **text**.
- A text field called **content**.

Create Newsletter content-type with two fields: title(text) and content(text)

Create a Controller to Send Newsletters

Create or edit `src/api/newsletter/controllers/newsletter.js`:
```js
"use strict";

const { createCoreController } = require('@strapi/strapi').factories;

module.exports = createCoreController('api::newsletter.newsletter', ({ strapi }) => ({
  async sendNewsletter(ctx) {
    try {
      const { id } = ctx.request.body;

      if (!id) {
        return ctx.badRequest("Missing required field: id");
      }

      // Fetch the newsletter from the database
      const newsletter = await strapi.entityService.findOne("api::newsletter.newsletter", id, {
        fields: ["title", "content"],
      });
      /*const newsletter = await strapi.documents("api::newsletter.newsletter", id).findOne({
        fields: ["title", "content"],
      });*/

      if (!newsletter) {
        return ctx.notFound("Newsletter not found");
      }

      const { title, content } = newsletter;

      // Fetch all subscribers
      const subscribers = await strapi.entityService.findMany("api::subscriber.subscriber", {
        fields: ["name", "email"],
      });
      /*const subscribers = await strapi.documents("api::subscriber.subscriber").findMany({
        fields: ["name", "email"],
      });*/

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

Create a Custom Route to Send Newsletters

`src/api/newsletter/routes/custom-routes.js`

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

Restart Strapi

Test API

```shell
curl -X POST http://localhost:1337/api/send-newsletter \
     -H "Content-Type: application/json" \
     -d '{
           "id": 1
         }'

```

## Automatic send when published

Create a new file for lifecycle hooks in your Newsletter model:

Create `src/api/newsletter/content-types/newsletter/lifecycles.js`

```js
// src/api/newsletter/content-types/newsletter/lifecycles.js
module.exports = {
  async afterCreate(event) {
    const { result } = event;
    if (result.publishedAt) {
      await sendNewsletterToSubscribers(result.id);
    }
  },
  
  async afterUpdate(event) {
    const { result, params } = event;
    // Check if the newsletter was just published
    if (result.publishedAt && (!params.data.publishedAt || params.data.publishedAt === result.publishedAt)) {
      await sendNewsletterToSubscribers(result.id);
    }
  },
};

async function sendNewsletterToSubscribers(id) {
  try {
    // Fetch the newsletter from the database
    const newsletter = await strapi.entityService.findOne("api::newsletter.newsletter", id, {
      fields: ["title", "content"],
    });

    if (!newsletter) {
      console.error("Newsletter not found");
      return;
    }

    const { title, content } = newsletter;

    // Fetch all subscribers
    const subscribers = await strapi.entityService.findMany("api::subscriber.subscriber", {
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
        subject: title,
        htmlContent: `<p>Dear ${subscriber.name},</p><p>${personalizedContent}</p>`,
      });
    }

    console.log(`Newsletter sent to ${subscribers.length} subscribers`);
  } catch (error) {
    console.error("Email error:", error);
  }
}
```

Test

The lifecycle hooks will trigger either when:

- A new newsletter is created with published status
- An existing newsletter is updated to published status

Check email

## Update subscribers collection using curl POST request

Enable Public Permissions (Create, FindOne, Find)

Test using Curl
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

Verify
```shell
curl -X GET http://localhost:1337/api/subscribers
```

## Create minimal subscribe form in (name, email) in Next.js

Create Next project

Make directory `frontend`
```shell
mkdir frontend
```

Install `react`, `react-dom`, and `next` as npm dependencies.
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
    "next": "^14.2.15",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
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

## Update subscribers collection using Next form (Connect Next to Strapi, )

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

## Interlude

- Go switch and replace with Koa.js switch and replace with Strapi middleware, custom endpoint, custom controller

Create collection called `Human` with a text type called `name` inside Strapi.

Give a few entries `Bollar`, `Craig`, `Paul` etc.

Give it public access

Create simple input form using html that submits the name, `name`.

```html
<body>
    <form action="#" method="post">
      <label for="name">Enter your name:</label>
      <input type="text" id="name" name="name" required />
    </form>
</body>
```
