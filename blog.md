# Cross channel notifications with Knock, Resend, and Next.js

Product led growth is all about nudges.
Create a well-defined onboarding path within your product, support users' journey with multiple notificiation channels that nudge them back onto that path, then use data to measure the effectiveness of these channels, the path itself, and the product value story you're telling.

Measure, optimize, and repeat.

The "nudge" part is essential, and more effective with multiple notification channels within your product.
[Knock](https://knock.app/) simplifies how quickly organizations can implement PLG strategies by allowing them to bootstrap the kind of full-fledged, multi-channel notification systems they those strategies require.

In this post, we'll take a look at one of Knock's example applications - a Node.js app written with [Blitz.js](https://blitzjs.com/) and [Next.js](https://nextjs.org/) - to see how an existing in-app notification system (using [Knock's Node.js SDK](https://github.com/knocklabs/knock-node) and [React in-app feed components](https://github.com/knocklabs/react-notification-feed)) can be adapted to include email notifications with [Resend](https://resend.com/).

## Setting up a demo application

First, let's start off by setting up a demo application we can use to initialize in-app notifications using Knock.
Later on, we'll add Resend email as a [Knock Channel](https://docs.knock.app/concepts/channels) to those workflows.

1. First, [create a Knock account](https://dashboard.knock.app/signup) if you haven't and log into the [Knock dashboard](https://dashboard.knock.app/).
1. Clone the demo app locally

   ```bash
   git clone git@github.com:knocklabs/knock-node-example-app.git knock-demo && cd knock-demo
   ```

1. Create a file for your local environment variables:

   ```bash
   touch .env.local
   ```

1. We'll need a local PostgreSQL instance to run the app. Add a `docker-compose.yaml` at the root of repository containing the following:

   ```bash
   # docker-compose.yaml
   version: "3.8"

   services:
       db:
           image: "postgres:latest"
           ports:
               - 5432:5432
           environment:
               POSTGRES_USER: johnhammond
               POSTGRES_PASSWORD: wehaveatrex
   ```

1. Start Docker, and spin up the PostgreSQL service:

   ```bash
   docker compose up -d
   ```

   > [!NOTE]
   > Don't forget to run `docker compose down` when you're finished with this example!.

1. In the `.env.local` file you created, add a `DATABASE_URL` variable to connect to the service:

   ```bash
   # .env.local
   DATABASE_URL=postgresql://johnhammond:wehaveatrex@localhost:5432/postgres
   ```

1. [Install the Knock CLI](https://docs.knock.app/cli):

   ```bash
   # Homebrew
   brew install knocklabs/tap/knock
   # Or NPM globally
   npm install -g @knocklabs/cli
   ```

1. Authenticate the CLI with a service token. From the Knock dashboard, visit **Settings** > **Service tokens** and create a new token. To simplify things, I copied my service token to a `~/.config/knock/config.json` file:

   ```json
   # ~/.config/knock/config.json
   {
       "serviceToken": "MY_SERVICE_TOKEN"
   }
   ```

   Then run `knock whoami` to authenticate.

1. Your account on Knock comes with two additional identifying keys - one public, and one secret. In our _non-committed_ environment variable file, add those two keys (which can be found under **Development** > **API Keys**):

   ```bash
   # .env.local
   DATABASE_URL=postgresql://johnhammond:wehaveatrex@localhost:5432/postgres
   KNOCK_API_KEY=<Your Knock Secret Key>
   BLITZ_PUBLIC_KNOCK_CLIENT_ID=<Your Knock Public Key>
   ```

1. To run the demo application, we need to define two channels in our Knock dashboard. Visit **Integrations** > **Channels** and use **+Create channel** to create two channels:

   - Type **In-app** titled `in-app-feed`
   - Type **Email** > **Postmark** titles `postmark`

   > [!NOTE]
   > We will replace this Postmark email channel with one for Resend once we've finished setting up workflows for the demo app.

1. Now the the `in-app-feed` channel has been defined in Knock, we can provide the demo application with it's unique **Channel ID** to our environment file:

   ```bash
   # .env.local
   DATABASE_URL=postgresql://johnhammond:wehaveatrex@localhost:5432/postgres
   KNOCK_API_KEY=<Your Knock Secret Key>
   BLITZ_PUBLIC_KNOCK_CLIENT_ID=<Your Knock Public Key>
   BLITZ_PUBLIC_KNOCK_FEED_ID=<in-app-feed Channel ID>
   ```

1. With the channels in place, we need to define three workflows in Knock:

   - **Welcome**: when a new user joins our demo "forum" application, they should receive a welcome email and in-app notification.
   - **New comment**: when another user leaves a comment in the "forum", other users should receive in-app notifications, then an email after a certain delay when the workflow is triggered.
   - **New asset**: when a new asset is added to the forum, a user should once again receive an in-app, then an email notification of that event.

   Rather than go through the steps of manually defining these workflows, the demo repository leverages that fact that workflows can be [_pulled_](https://docs.knock.app/cli#workflow-pull) and committed to simplify the process. You can find the three workflows (`new-asset`, `new-comment`, and `welcome`) within the [`knock/workflows`](https://github.com/knocklabs/knock-node-example-app/tree/main/knock/workflows) subdirectory in the example repo.

   Now that you're authenticated through the CLI, use the below command to push the committed workflows to your Development environment on Knock:

   ```bash
   knock workflow push --workflows-dir=knock/workflows
   ```

   > [!NOTE]
   > Using the demo application as-is, I ran into [a small error](https://github.com/chadwcarlson/knock-node-example-app/pull/1/files#diff-7aacaa4c69c1195ff537dbae91c5056c41f45ce210cfccd9de1256d308ee42dfR18) using `workflow push`, related to the liquid template used to pluralize text on an e-mail's CTA button when more than one comment has not been seen by the user.
   > For the time being, I have replaced that line with what is seen in the linked commit above until a proper issue is opened.

1. With our environment variables and workflows in place, we can finally run our migrations and start the demo application:

   ```bash
   # Run migrations
   yarn blitz prisma migrate dev
   # Seed the database
   yarn blitz db seed
   # Run the development server
   yarn blitz dev
   ```

   > [!NOTE]
   > Using the demo application as-is, I ran into [another error](https://github.com/chadwcarlson/knock-node-example-app/pull/1/files#diff-5e4bf942daca7c0e27958ab33fc3c8de5a3a79643e809bca1177e15d8ed0a166R56) when running `yarn blitz dev` to run a local development server. As written, the application was not able to retrieve the default workflow preferences for the three defined workflows, causing the app to fail. Commenting out those failing sections allows the local server to run correctly, but indicates an issue that needs reporting.

Following the initialization and authentication steps above, we'll get a functioning in-app notification system for a demo application based on the characters from Jurassic Park.

<p align="center">
<img src="blog_img/workflows.png">
</p>

From here, you can **Run a test** on the workflow from the Knock dashboard, or log into the server (see `db/seeds.ts` for seeded user accounts) from another browser to see it in action.

## Resend for delayed email notifications

We've created workflows in the previous steps we'd like to connect a Resend email integration to.

Let's start with the **Welcome** workflow, which will provide a welcome message in the in-app notification feed and (to begin with) send a message to the Postmark email service to email that new user a welcome email.

<p align="center">
<img src="blog_img/welcome.png">
</p>

1. Go to **Integrations** > **+Create channel** > **Resend**. Name the integration `resend`, and save it.
1. Create an account, if you haven't already done so, on [Resend](resend.com).
1. Visit [https://resend.com/api-keys](https://resend.com/api-keys) and create an API key, at least with _Sending access_.
1. On the Knock dashboard, click on the Resend channel you just created, then on the **Manage conifiguration** button for the **Development** environment.
1. Fill out the form, first with the API key you copied from Resend. Fill out (at least) the **From email address** in the form using an address associated with a domain name you own - you will need to add configuration from Resend to your DNS provider in the next step for this domain.

<p align="center">
<img src="blog_img/resend_channel.png">
</p>

6. Visit [https://resend.com/domains](https://resend.com/domains), and then click **+ Add domain**. Add your domain to the field provided, making sure that your resulting configuration results in an email-specific subdomain, such as `send.YOUR_DOMAIN.com`.
7. Once you have chosen a region and saved the domain, Resend will provide three DNS records you will need to add to your DNS provider (one MX, and two TXT records). On Google Domains, for example (where I have my personal domain registered), you can go to **DNS** > **Manage custom records** and add the three records to your domain's configuration.
8. Save the configuration on both your DNS provider and on Resend, then Verify the configuration on Resend (which can take a few minutes).

<p align="center">
<img src="blog_img/resend_domain_config.png" width="800px">
</p>

9. Now that Resend has been connected to your DNS provider, it's time to actually connect the channel to the **Welcome** workflow. Visit the **Welcome** workflow, click on the **Postmark** step, and delete the step.
10. Add a new step, and select Email from the list of Channel types.
11. In the drop-down under **Select a channel**, choose **Resend**. Click **Commit to Development**, provide a commit message, and **commit** the change.

<p align="center">
<img src="blog_img/add_resend.png">
</p>

12. After the steps above have been completed, you can personalize the look of the resulting email by visiting **Settings** > **Branding** to set up the basic branding of your default email template, as well as the content message template by clicking **Edit template** under the **Resend** step within the **Welcome** workflow.

If the demo application is still running locally, you can test out this newly revised workflow:

1. Logout.
1. Visit [http://localhost:3000/signup](http://localhost:3000/signup) to create a new account with your own email.

Once you have done so, you should receive an email in your inbox, verifying that the integration has been set up and is working as expected.

<p align="center">
<img src="blog_img/welcome_email.png">
</p>

Just as we have defined in the workflow, each new user will get a welcome message within the application and an email like the one above.

## Extending to other workflows

With the integration set up, it's trivial to setup Resend email notifications on any other workflow we have defined in Knock.
We can similarly remove the Postmark email step we pushed to the **New comment** workflow with Resend, and quickly ensure that (in this case) comments not read in the in-app notification feed within one minute by the user are sent via email.

<p align="center">
<img src="blog_img/new_comment.png">
</p>

If you notice, most of the steps in this post were dedicated to setting up the demo application to teach you how to enable a Resend integration through Knock.
That is, the integration itself only required a few settings registered either within Knock, Resend, or your DNS provider to add a whole new channel to your organization's notification system.

Imagine if we had to build (and maintain) all of this instead from scratch?
Frankly, I'd rather spend my time developing new features for my application, and thinking about how I can optimize notifications to drive our product goals.

Good thing we've got Knock!
