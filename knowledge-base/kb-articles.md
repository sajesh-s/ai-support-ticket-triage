# Cloudnest Help Center (fictional knowledge base)

All content is invented for this demo project. "Cloudnest" is a fictional SaaS company.

---

## KB-01: Password reset link expired
Reset links are valid for 30 minutes and only the most recent link works. Ask the user to request a new link and open it within 30 minutes, in a private browser window. If many users at one company are affected at once, escalate to engineering as a possible incident.

## KB-02: Unlocking a locked-out user
After 5 failed attempts an account is locked for 15 minutes. An admin can unlock it immediately under Settings > Users > select user > Unlock account.

## KB-03: Two-factor (SMS) codes not arriving
Check that the phone number includes the country code (+31 for the Netherlands). Users can switch to an authenticator app under Profile > Security. If several users are affected, escalate: it may be an SMS provider issue.

## KB-04: Roles and permissions
The Manager role does not include Finance access by default. Admins can add it under Settings > Roles > Manager > Permissions > Finance (view).

## KB-05: Setting up SSO (Azure AD / Entra ID)
SSO is available on the Enterprise plan. The customer's IT team needs to create an Enterprise Application in Azure, then share the Metadata URL. Admin enters it under Settings > Security > Single sign-on. Test with one user before enforcing SSO for everyone.

## KB-06: Duplicate charges and refunds
Duplicate card charges are usually a pending authorisation that disappears within 5 working days. If both are settled, support can issue a refund from the billing console. Refunds take 5 to 10 working days.

## KB-07: VAT on invoices
Dutch customers are charged 21% VAT. If 0% appears, the account's billing country or VAT number is wrong. Correct it under Billing > Company details; support can reissue the invoice.

## KB-08: Cancelling a subscription
Admins cancel under Billing > Subscription > Cancel. Access continues until the end of the paid period. No refunds for partial months.

## KB-09: Adding users and pricing
Add users under Settings > Users > Invite. Each extra user costs EUR 9 per month (Starter), EUR 15 (Business) or according to contract (Enterprise).

## KB-10: Changing the billing email
Admins can change the invoice recipient under Billing > Company details > Invoice email.

## KB-11: Exporting data
Most lists have an Export button (top right) that downloads CSV, which opens in Excel. Large exports are emailed when ready.

## KB-12: Deleting projects
Archive a project under Project > Settings > Archive. Permanent deletion is possible after archiving and cannot be undone.

## KB-13: CSV import missing rows
Rows are skipped when required fields (SKU, name, price) are empty or duplicate SKUs exist. Download the import log under Imports > History to see skipped rows.

## KB-14: API errors and rate limits
Rate limit: 100 requests per minute per API key. HTTP 429 means the limit is reached; use batching and retry with backoff. HTTP 500 errors are server-side: collect request IDs and timestamps and escalate to engineering.

## KB-15: Webhooks not delivering
Webhooks are disabled automatically after 20 consecutive failed deliveries. Check Settings > Developers > Webhooks for the status and re-enable after fixing the endpoint.

## KB-16: Performance issues
Slow dashboards and search are often caused by very large date ranges or filters. Ask for the account, page, time of day and a browser HAR file, then escalate to engineering with details.

## KB-17: Calendar sync duplicates
Duplicates occur when the calendar is connected twice. Disconnect all calendars under Integrations > Calendar, then reconnect once.

## KB-18: Integrations available
Native integrations: Google Calendar, Outlook, Exact Online, Mollie, Zapier. Mailchimp is available through Zapier.

## KB-19: Security and compliance documents
The SOC 2 Type II report and security questionnaire answers are shared by the account manager under NDA. Route these requests to the Account team.

## KB-20: GDPR deletion requests
Admins can delete a person's data under Customers > select customer > Delete personal data. Deletion is completed within 30 days, within the GDPR deadline. Support can confirm completion in writing.

## KB-21: Suspicious logins
Treat as a security incident. Advise the admin to reset passwords for affected accounts and enforce two-factor login immediately. Escalate to the Security team right away.

## KB-22: Outages
Check status.cloudnest.example. For a full outage affecting a customer, escalate immediately as a P1 incident and update the customer every 30 minutes.
