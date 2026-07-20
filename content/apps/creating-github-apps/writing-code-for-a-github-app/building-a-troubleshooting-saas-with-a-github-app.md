---
title: Building a troubleshooting SaaS with a GitHub App
shortTitle: Build a troubleshooting SaaS
intro: 'Plan a software as a service app that helps teams resolve issues by using a {% data variables.product.prodname_github_app %}, webhooks, and the REST API.'
versions:
  fpt: '*'
  ghes: '*'
  ghec: '*'
category:
  - Write code for a GitHub App
---

## Introduction

You can build a software as a service (SaaS) app that helps teams resolve problems reported in issues. For example, your app can watch for new issues, ask for missing information, add labels, create a support workflow in an external system, and report progress back to the issue.

A {% data variables.product.prodname_github_app %} is a good way to build this type of integration because each customer can install the app on the repositories that they choose. Your service can then use installation access tokens to call the REST API only for the repositories that the customer granted access to.

This article describes an architecture for a troubleshooting SaaS app. It does not include a full production application, but you can use it to design your data model, permissions, webhook handling, and API calls before you write code.

## Example workflow

A troubleshooting SaaS app might use the following workflow:

1. A customer installs your {% data variables.product.prodname_github_app %} on one or more repositories.
1. Someone opens an issue that describes a problem.
1. {% data variables.product.prodname_dotcom %} sends an `issues` webhook event to your service.
1. Your service validates the webhook signature and stores the installation ID, repository ID, issue number, and delivery ID.
1. Your service analyzes the issue and decides whether to add a label, ask a clarifying question, or create a task in your support system.
1. Your service uses the REST API to update the issue, such as by adding labels or creating a comment.
1. When the support task changes state, your service comments on the issue or closes the issue when the problem is resolved.

## Choose app permissions and events

Request the least access that your app needs. For a basic troubleshooting workflow, configure your {% data variables.product.prodname_github_app %} with these repository permissions:

| Permission | Access | Why it is needed |
| --- | --- | --- |
| Issues | Read & write | Read issue details, update issue state, and create issue comments. |
| Metadata | Read-only | Required for all {% data variables.product.prodname_github_apps %}. |

Subscribe to these webhook events:

| Event | Why it is useful |
| --- | --- |
| `issues` | Start a troubleshooting workflow when an issue is opened, reopened, edited, assigned, labeled, or closed. |
| `issue_comment` | Continue the workflow when a user answers a question or adds more diagnostic details. |
| `installation` | Create, update, or remove tenant records when customers install, suspend, or uninstall your app. |
| `installation_repositories` | Keep repository access in sync when customers add or remove repositories from an installation. |

For more information about configuring permissions and events, see [AUTOTITLE](/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app) and [AUTOTITLE](/apps/creating-github-apps/registering-a-github-app/using-webhooks-with-github-apps).

## Design a tenant model

Store each installation as a separate tenant. Your data model should let you map API requests and webhook deliveries back to the customer installation that authorized the access.

For each installation, store:

* The installation ID.
* The account login and account ID.
* The repositories that the installation can access.
* Customer configuration, such as labels to apply, escalation rules, or external support queues.
* Whether the installation is active, suspended, or uninstalled.

Do not store installation access tokens long term. Installation access tokens expire, so your service should create a token when it needs to call the API. For more information, see [AUTOTITLE](/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app).

## Process webhooks safely

Your webhook endpoint should be fast and idempotent.

1. Verify the webhook signature before you trust the payload. For more information, see [AUTOTITLE](/webhooks/using-webhooks/validating-webhook-deliveries).
1. Use the `X-GitHub-Delivery` header as an idempotency key so that retries do not create duplicate comments or duplicate support tickets.
1. Store the payload details that you need for later work, then enqueue a background job.
1. Return a successful HTTP status quickly after the event is accepted.
1. In the background job, create an installation access token and call the REST API.

## Call the REST API

The REST API endpoints that you call depend on the workflow that your SaaS provides.

| Goal | REST API endpoint |
| --- | --- |
| Read the issue that triggered a workflow | [Get an issue](/rest/issues/issues#get-an-issue) |
| Ask for missing diagnostic information | [Create an issue comment](/rest/issues/comments#create-an-issue-comment) |
| Add a triage or support label | [Add labels to an issue](/rest/issues/labels#add-labels-to-an-issue) |
| Assign a support engineer | [Add assignees to an issue](/rest/issues/assignees#add-assignees-to-an-issue) |
| Close the issue when the problem is resolved | [Update an issue](/rest/issues/issues#update-an-issue) |

When you call the API, include enough information in comments for users to understand what your app did and what they should do next. Avoid posting secrets, private customer data, or internal support notes in public repositories.

## Handle rate limits and retries

A SaaS app can receive events from many installations at the same time. To keep your integration reliable:

* Queue webhook work and process it asynchronously.
* Partition work by installation ID so that one busy customer does not block other customers.
* Respect REST API primary and secondary rate limits. For more information, see [AUTOTITLE](/rest/using-the-rest-api/rate-limits-for-the-rest-api) and [AUTOTITLE](/rest/using-the-rest-api/best-practices-for-using-the-rest-api).
* Retry transient API errors with exponential backoff.
* Treat `404`, `403`, and `410` responses as signals to refresh installation or repository access before retrying.

## Secure customer data

Before you launch your SaaS app, review how you handle data for each customer.

* Encrypt private keys, webhook secrets, customer tokens, and external support credentials.
* Separate customer data by installation ID or another tenant boundary.
* Log webhook delivery IDs and API request IDs, but avoid logging issue bodies if they may contain sensitive information.
* Provide a way to delete customer data when an installation is uninstalled.
* Document what data your app reads, stores, and writes.

## Next steps

After you design the workflow, you can build a prototype that receives webhook events and comments on issues. For a step-by-step example of receiving webhook events with a {% data variables.product.prodname_github_app %}, see [AUTOTITLE](/apps/creating-github-apps/writing-code-for-a-github-app/building-a-github-app-that-responds-to-webhook-events).
