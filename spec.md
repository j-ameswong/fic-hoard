# Fic Hoard: Initial Product Specification

**Status:** Initial draft for discussion and repeated revision.

**Date:** 28 September 2026.

This document records the product direction and behaviours agreed so far. It is not an implementation plan or task breakdown. Detailed mobile interactions, per-site field contracts, the data model, API, and implementation choices remain open. Sections marked **TBD** or **working proposal** need further refinement.

## 1. Purpose and scope

Fic Hoard is a private, cross-site fanfiction library and reading tracker. It helps readers save stories, return to a saved reading location, identify unread updates, and rediscover stories through metadata, personal notes, and tags.

Readers continue reading on the original website. The first version stores metadata, chapter-index information where available, and personal library information; it does not archive fic text or provide an in-app reader.

The application must be mobile first, easy to use, and require as little effort as possible to import and maintain a library. The expected initial user base is small. Imports should use sensible, bounded timeouts without assuming that external sites will always respond promptly.

Each user's library is private. Social features, public libraries, recommendations between users, and other cross-user interactions are outside the planned scope.

Anyone should be able to clone the source and self-host the application, including running it without authentication in an explicit single-user mode.

## 2. Sources and story identity

Initial automatic-import targets are:

- Archive of Our Own (AO3).
- FanFiction.net.
- XenForo forums: SpaceBattles, Sufficient Velocity, and Questionable Questing.

Direct retrieval uses site-specific parsing with jsoup. FicHub provides an alternative metadata source when appropriate. Manual entry remains available when automatic retrieval cannot provide the required information.

A valid fic URL from a site supported by neither the application nor FicHub can be saved as a manual-only entry. Unsupported sites are not, by themselves, import errors.

Story identity is based on the source site and its work/thread identity where that can be established. Chapter URLs belonging to the same story resolve to one normalized fic URL and one entry per user. Titles and author names are not sufficient identifiers.

Cross-posts on different sites are separate entries. Automatic or manual cross-post merging is outside the initial scope.

Recognized non-fic pages, such as search results or author profiles, should prompt the user to provide a fic link. Invalid inputs are rejected with an explanation. The exact URL-validation and normalization rules are **TBD** per source.

## 3. Fic metadata and personal information

The metadata model should accommodate the following information when available. Inclusion here does not make a field mandatory for every source.

| Information | Requirements |
| --- | --- |
| Source | Site, source identity, and normalized fic URL |
| Title and authors | Support multiple authors and legitimate anonymous attribution |
| Summary | Source-provided story summary |
| Language | Preserve the reported language |
| Rating | Preserve the source's rating terminology; cross-site mappings remain TBD |
| Warnings | Distinguish explicit warnings, explicit no-warning declarations, a choice not to specify warnings, and unavailable information |
| Fandoms | A type of tag, exposed as a first-class field supporting multiple fandoms |
| Other source tags | Preserve useful categories such as relationships, characters, and freeform tags |
| Word count | Unknown and zero are distinct values |
| Chapters | Published chapter count, chapter index where available, and planned total where reported; published and planned totals are distinct |
| Fic status | Ongoing, complete, unknown, or hiatus |
| Dates | Source publication and update dates, separate from application fetch and library dates |
| Site-specific metrics | Preserve names and meaning, such as kudos, favourites, or reactions; do not collapse them into one universal popularity measure |

Each user's library entry includes:

- Reading status.
- A saved resume URL.
- Completed reading progress.
- A memorable personal note, for example a plot reminder for future rereading.
- Personal tags reusable across that user's library.
- Private manual metadata and corrections where needed.

Imported tags and personal tags have distinct ownership. Refreshes must not overwrite personal tags, notes, progress, or manual metadata overrides. Manual metadata and restricted information must not become another user's metadata through a shared cache.

Public source metadata may be reused internally within an installation to avoid duplicate requests. This does not make users' libraries discoverable to one another.

Store enough provenance to distinguish direct-source data, FicHub data, and manual values. Record fetch attempts and successful observations separately from the source's own update date. A recent fetch time does not necessarily mean the underlying metadata is current.

The physical representation of common fields, typed tags, site-specific metrics, provenance, and overrides is **TBD**.

## 4. Statuses

Fic status and reading status are independent.

| Fic status | Meaning |
| --- | --- |
| Ongoing | The fic is understood to be in progress |
| Complete | The source or a personal override identifies the fic as complete |
| Unknown | Its publication status cannot currently be established |
| Hiatus | An explicit source indication or personal override identifies a hiatus |

A long period without updates does not automatically imply hiatus. A failed refresh does not replace a previously known fic status with unknown.

| Reading status | Meaning |
| --- | --- |
| To read | Saved for future reading |
| Reading | The reader is working through the fic |
| Caught up | The reader has marked all chapters in the accepted latest snapshot as completed |
| Read | The reader has explicitly marked the fic as read |
| Dropped | The reader has chosen to stop reading |

Source completion never automatically marks a fic as read. Read does not require the source fic to be complete. New story chapters change Caught up to Reading, while Read and Dropped remain explicit user choices.

## 5. Initial import and editing

A valid new fic link is saved immediately while initial enrichment runs. The application attempts direct metadata retrieval, uses FicHub where appropriate, and offers manual entry when automatic providers cannot supply usable information.

During initial enrichment, **only notes are editable**. Reading status, personal tags, progress, and metadata editing remain locked until that initial process finishes. Initial enrichment must end on success, an accepted partial result, or manual fallback; errors must not leave the editor locked indefinitely.

When initial enrichment finishes, the application assigns a conservative initial reading status:

| Link information established during import | Initial reading status |
| --- | --- |
| Base fic URL or first chapter | To read |
| Recognized chapter or story threadmark after the first | Reading |
| Ambiguous forum page, unrecognized post, or insufficient information to establish chapter position | To read |

A recognized later chapter still implies only Reading even if it is the final chapter of a completed fic. Caught up, Read, and Dropped require explicit user selection.

The inferred status becomes editable without an additional confirmation step. Inference runs once for a new entry. Reimports and routine refreshes do not rerun it.

Routine background refreshes leave editing available and preserve edits made while they run. Manual-only entries skip unavailable automatic providers and open the manual editor immediately.

## 6. Resume links and duplicate imports

The pasted URL is saved as `resume_url`, independently of the normalized fic URL. Preserve useful chapter information, query parameters, and fragments in that saved link.

| Reading status | Primary reading action opens |
| --- | --- |
| To read or Read | Normalized fic URL |
| Reading, Caught up, or Dropped | Saved resume URL |

The resume URL belongs to the user's library entry. It is retained across status changes, including statuses that currently use the base fic URL.

Reimporting an existing fic replaces its resume URL only. It does not reset notes, tags, reading status, completed progress, or manual corrections. Resolving a chapter label from the saved link does not imply that the chapter has been completed.

Opening a link, changing completed progress, refreshing metadata, or marking Caught up/Read does not automatically rewrite or delete the resume URL. In particular, detecting a new chapter does not automatically move the saved link to that chapter.

## 7. Recording reading progress

Use a dropdown of chapters or story threadmarks when a reliable index is available. Show chapter numbers and titles where available. Provide numeric entry for the last completed chapter when no usable list is available.

Chapter identity and chapter position are separate concepts. AO3 chapter IDs do not encode chapter numbers. Its chapter index can map chapter IDs and URLs to their ordered positions and titles. The adapter should use that mapping to support the dropdown and identify the saved link's chapter. [AO3 chapter index](https://github.com/otwcode/otwarchive/blob/master/app/views/works/navigate.html.erb), [chapter numbering](https://github.com/otwcode/otwarchive/blob/master/app/models/chapter.rb).

Retain the identity of the selected chapter/post where available, together with enough information about its order when progress was recorded. The exact representation of checkpoints and chapter snapshots is **TBD**.

The saved reading location and completed progress may legitimately differ. For example, the resume link may open chapter 12 while the reader has completed chapter 11. Fetching chapter 12 cannot establish whether the reader has finished it.

Where the saved link resolves to a chapter, offer an optional **Read through this chapter** action. This explicitly updates completed progress without requiring a separate search through the dropdown. Ordinary URL import or reimport does not take this action automatically.

The interface should distinguish the saved reading location from the last completed chapter. Exact controls, labels, numeric validation, and placement are **TBD**.

### Marking Caught up or Read

Both actions follow the same progress-update flow:

1. Request a metadata refresh to establish the latest published chapter count.
2. Obtain the chapter/threadmark index where available.
3. Record all published chapters in the accepted refresh result as completed. Record the final chapter identity where it is reliably available.
4. Save the requested reading status and completed checkpoint together.
5. Display the result, such as **Caught up through chapter 19**, with an Undo action.

The count is the number of published story chapters, not the planned final total or the number of forum replies. An otherwise partial metadata result can support this action if it establishes a usable current chapter count.

On success, save immediately without an extra confirmation. The action means that all chapters reported by that refresh are read, including any newly discovered chapters. A surprise update between reading on the source site and performing this action can be corrected with Undo.

The checkpoint is tied to that result. Later refreshes must not silently extend completed progress. If chapter 20 is subsequently detected after marking through chapter 19, completed progress remains at 19.

Undo reverses the action's personal status/progress changes while retaining refreshed source metadata. Existing rules for newly discovered unread chapters still apply. Undo does not change the resume URL.

If the refresh fails or cannot establish a current count, preserve the existing reading status and checkpoint. Offer Retry, an explicit choice to use the last known count, or manual number entry. Do not silently treat stale fallback data as verification of the latest count.

Manual-only entries use the same explicit count-confirmation or number-entry fallback without attempting unsupported automatic retrieval.

## 8. XenForo chapter handling

Use story threadmarks rather than ordinary reply counts to establish reading units and detect updates. Default to the main-story category; side stories, omake, informational posts, and discussion replies must not silently inflate that count.

Handle chapter-index pagination and retain stable post/threadmark identities where available. An incomplete index cannot establish that omitted chapters were deleted. If no reliable story index or count can be established, support manual progress.

Forum-specific capabilities can differ even when adapters share parsing code. Exact category handling and the contracts for SpaceBattles, Sufficient Velocity, and Questionable Questing remain **TBD**. Threadmarks Pro documents categories, index pagination, and reader-mode capabilities relevant to this work. [Threadmarks Pro documentation](https://xenforo.com/community/resources/threadmarks-pro.6615/).

## 9. Import and refresh contracts — working proposal

Each supported site needs an explicit contract defining mandatory, optional, and conditionally available fields. The exact field matrices are **TBD**.

The proposed contract distinguishes:

- **Minimum identification:** enough evidence that the response belongs to the intended fic, rather than an error, login, or unrelated page.
- **Expected metadata:** what a normal accessible page should provide for that site and page type.
- **Feature requirements:** information needed for a chapter dropdown, unread calculation, or Caught up/Read action.

Missing information needs a reason. Legitimately absent, unsupported by a provider, inaccessible, incompletely retrieved, and unexpectedly unparseable are different conditions. For example, an intentionally empty optional summary does not by itself make a fetch partial.

Data quality and recovery action should be evaluated separately. Provider capabilities also matter: FicHub should not be repeatedly retried for a field it does not supply, even if the direct site normally supplies it.

| Fetch outcome | Proposed handling |
| --- | --- |
| Expected metadata retrieved | Accept the result |
| Optional information legitimately absent | Accept; this need not count as partial |
| Useful information retrieved with unexpected omissions | Accept partial data and decide recovery based on the cause |
| A recoverable portion, such as an index page, fails | Preserve usable data and retry within a bounded budget |
| Temporary network/server error | Use bounded retries with delay |
| Rate limit | Respect the provider's retry timing |
| Access restriction or challenge | Try FicHub where eligible, then manual entry if needed |
| Automatic providers exhausted | Offer manual entry prefilled with usable retrieved information |
| Valid fic URL unsupported by all providers | Create a manual-only entry |
| Invalid input or recognized incompatible page type | Explain the problem and request a valid fic link |

An unexpected missing field does not automatically justify an immediate retry. A transient request failure may recover, while retrying the same page with a broken parser is unlikely to help. A partial result requiring no recovery retry remains eligible for ordinary scheduled refreshes.

Initial enrichment has an overall deadline as well as bounded attempts. When that deadline is reached, release the editing restriction and expose usable partial data or manual fallback. Later recovery attempts must preserve personal edits.

FicHub is an alternative metadata provider, not a guarantee of access or freshness. Its API requests must be sequential, identify the application with contact information, and respect rate limits and `Retry-After`. Its API is documented as unstable, and its cached content may be old. [FicHub API](https://fichub.net/api), [FicHub caching information](https://fichub.net/).

The general retrieval path is direct source, then FicHub where eligible, then manual entry. Exact provider eligibility, timeouts and retry budgets, reason codes, and all per-site required-field lists remain **TBD**.

## 10. Refreshing and reconciling metadata

Regular background refreshes should keep unfinished fics current enough to help readers identify unread updates. Refresh demand for the same public source record should be shared within an installation rather than multiplied by the number of readers tracking it.

Initial scheduling direction, to be tuned later:

- Roughly daily checks for unfinished stories actively being read or marked Caught up.
- Less frequent checks for backlog, dormant, or hiatus entries.
- Occasional reconciliation and explicit manual refresh for completed stories.
- No automatic metadata refreshes for manual-only entries.

Requests need per-site limits and backoff. Failed checks retain the last good information and expose when metadata was last successfully checked. Exact schedules and job implementation are **TBD**.

A new story chapter is different from a metadata change. Changes to kudos, tags, ordinary forum replies, or similar information must not create unread-chapter alerts.

Refreshes follow these preservation rules:

- Failure to retrieve a field does not erase its previous value. A confirmed empty value is distinct from a retrieval failure.
- Stale cached values must not silently replace newer reliable observations.
- Genuine, verified source corrections remain possible; chapter counts are not assumed to increase forever.
- Personal overrides, notes, tags, reading progress, and resume links survive refreshes.
- Source unavailability does not delete the library entry or replace its fic status. Access failures and 404 responses are not sufficient grounds to discard saved information.

For chapter changes, compare stable identities and sufficiently complete observations. Reordering should not be mistaken for newly published chapters. Preserve the checkpoint and flag it for review if its chapter disappears rather than silently assigning it to another chapter or resetting progress.

Detailed reconciliation of inserted, deleted, and reordered chapters, including how uncertainty is displayed, remains **TBD**. The reader's completed checkpoint must retain enough historical context to avoid interpreting a changed ordinal as evidence of reading a different chapter.

## 11. Mobile screens and interactions — TBD

The application must support these product needs:

- A personal library with easy access to continuing reading and finding new chapters.
- URL import and manual entry with clear loading, partial-result, and fallback behaviour.
- Fic details and editing of personal information and progress.
- Search across titles, authors, summaries, and personal notes.
- Filtering by reading status, fic status, source, fandom, word count, and source/personal tags.
- Settings for the supported account and library features.

The screen structure, navigation, layouts, exact search/filter controls, filter combinations, sorting, quick-action placement, and responsive interactions are **TBD**. The behaviours defined elsewhere in this document constrain those designs without prescribing a layout.

## 12. Accounts and self-hosting

Authenticated installations support:

- Google SSO/sign-in.
- Account creation with email and password.
- Email verification for email/password accounts.
- Password recovery.

The detailed flows, account-linking rules, registration policy, email delivery configuration, and other account lifecycle policies are **TBD**. Authentication and authorization must preserve each user's private library.

Self-hosting must also support an explicit authentication-disabled, single-user mode. Requests in this mode use one persistent local library owner; they do not create a separate library for every visitor. Anyone with access to that installation can use that shared local library.

Single-user operation should use the same library behaviours and ownership concepts as authenticated operation. It must not require Google authentication or an email service. Mode configuration and migration between modes are **TBD**.

Self-hosting documentation should cover installation, configuration, persistent storage, and backup/restore. Deployment details remain open.

## 13. Import/export scope and deferred features

The initial focus is single-URL import and manual entry. Browser-bookmark parsing and bulk import are deferred. Future bulk formats under consideration are browser bookmark HTML, pasted URL lists, CSV, and Markdown; none has a finalized format or mapping contract yet.

Keep a simple library export in the initial scope so users can retain their metadata, URLs, notes, tags, statuses, progress, and manual corrections. Its format is **TBD**.

The initial scope excludes cross-post merging, story-text archiving, an in-app reader, social features, and browser integrations that automatically observe reading activity or scroll position.

## 14. Technical design and task breakdown — TBD

The planned stack is Spring Boot for the backend and SvelteKit for the frontend, with jsoup for supported direct HTML extraction and FicHub as an alternative metadata provider. PostgreSQL is the current database direction, to be revisited during technical design.

No physical database schema, API contract, authentication library, job system, deployment architecture, or actionable task breakdown is defined by this draft. Those decisions should follow further product refinement.

The next iterations should resolve the open site contracts, mobile interactions, account flows, and metadata reconciliation rules before turning this specification into implementation tasks and formal acceptance tests.
