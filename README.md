# cue

![cue](assets/cue-mark-512.png)

The plugin uses the Cue product mark. HumanMeetsAI is the publisher; its corporate logo is not the plugin icon.

Create, edit, and publish animated presentations and print-ready documents on cue from a conversation. The plugin pairs cue's authoring skill with its hosted MCP connector, so Claude can work on your drafts, check the build, publish a link, revise an existing deck, and inspect feedback available to your account.

## Connect your account

A cue account with the relevant authoring or sharing permissions is required. The connector uses OAuth at https://cue.humanmeetsai.com/mcp. Sign in to cue in the browser and approve the connection. No API key, access token, or password belongs in this plugin's files.

On claude.ai and in Cowork, add or connect cue from the plugin's Connectors tab. In Claude Code, the plugin registers the remote server; use `/mcp` to complete its sign-in when needed. Account entitlements determine which cue features and limits are available. See [cue](https://cue.humanmeetsai.com) for the service and [connection help](https://cue.humanmeetsai.com/connect) for setup.

## Try it

- “Create a five-slide cue presentation about our quarterly product update for a ten-minute team meeting. Keep the first result as a draft.”
- “Show my cue decks and revise the closing slide of the one titled Quarterly Update.”
- “Publish this reviewed cue deck with the audience access I choose and give me its link.”

Describe the audience, language, length, and desired outcome. Review generated content before publishing. Publishing, changing sharing, unpublishing, and deletion affect content in your cue account; ask for the specific action you want.

## What the plugin contains and does

The package contains a Markdown authoring skill, a JSON connection declaration, listing metadata, and a cue brand image. It has no hooks, local MCP executable, dependency installation, or bundled runtime. Its remote MCP tools run on cue's service. Tool names and availability can change with the hosted service and your account permissions.

The skill guides Claude through creating or reading a server-held draft, editing source, submitting it to cue's validation/build service, checking the job, and returning its preview or published link. cue validates and renders the source on its servers. The skill may also guide account-authorized sharing, feedback, reader activity, talk, asset, and document workflows.

The skill uses `<CUE_URL>` for https://cue.humanmeetsai.com. It works with that site's template/library references and documentation. This package uses its bundled instructions; instruction updates arrive through a new plugin package version. It does not replace itself with remotely served behavioral instructions.

For a user-requested image, cue can fetch a public URL that the user or Claude selects and store the uploaded asset with the deck. Private files are uploaded only through cue's upload flow. The skill references the public KaTeX supported-command documentation at https://katex.org/docs/support_table for formula syntax. The plugin does not collect browser history or import an address book.

## Data and access

The connector sends tool requests to cue and returns the account-authorized results to Claude. Depending on the requested workflow, these can contain deck/document source and uploaded assets, titles and revision metadata, sharing recipients, notes and feedback, reader activity, or talk/transcript information. cue stores content and service records to provide the requested functionality. Claude's handling of the conversation is governed by your Anthropic account and its terms.

cue's [privacy policy](https://cue.humanmeetsai.com/privacy) describes the service's collection, processors, retention, and deletion. It states that cue does not use user content to train models or sell or rent personal data. Reader records normally expire after 180 days and server error records after 90 days. Other content generally remains until deleted or the account is deleted; minimal abuse records, suppression records, and normal backup/cache lifetimes are exceptions. Published or downloaded copies cannot always be recalled. Use [your account](https://cue.humanmeetsai.com/account) for account deletion and [support](https://cue.humanmeetsai.com/support) for data requests.

## Support and terms

- [Documentation](https://cue.humanmeetsai.com/docs/mcp)
- [Support](https://cue.humanmeetsai.com/support)
- [Privacy](https://cue.humanmeetsai.com/privacy)
- [Terms of service](https://cue.humanmeetsai.com/terms)
- [Acceptable use](https://cue.humanmeetsai.com/acceptable-use)

The service is operated by HUMANMEETSAI PTE. LTD., Singapore (UEN 201901706Z), formerly DEYI MEDICAL CONSULTING PTE. LTD. The legal company name changed on 11 November 2026. The [Cue Plugin License](LICENSE) permits installing and using this integration with Cue and using its included icon to identify Cue within the integration. Other rights are reserved. The hosted service remains subject to its separate terms.
