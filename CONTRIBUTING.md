# Contributing to the MCP social media guide

This repo accepts three kinds of contribution: a new tutorial, a fix to an existing one, and a correction to a client or server detail that has changed since it was written. All three go through a pull request. Nothing gets merged that the reviewer could not run.

## Proposing a tutorial

Open an issue before you write anything. A tutorial takes a day to write and ten minutes to reject, so the issue saves you the day. Include:

1. **The working title and the primary keyword.** One keyword per tutorial. If it overlaps an existing page, say which one and why yours is not a duplicate.
2. **The reader's starting state.** What do they already have installed, configured and authorized?
3. **The finishing state.** One sentence, concrete. "The reader has a scheduled draft on two platforms and can cancel it" is a finishing state. "The reader understands scheduling" is not.
4. **The command list.** A rough sequence of what the reader will actually type or click. If there are fewer than five commands, it is probably a section inside an existing tutorial rather than a new one.
5. **Which server you will use for the worked example**, and whether the technique generalizes to others. Vendor specific examples are fine. Vendor specific claims of superiority are not.

A maintainer replies within a week: accept, decline, or fold it into an existing page.

## House style

- **No marketing language.** Banned outright: seamless, robust, leverage, unlock, game changer, revolutionize, effortless, powerful. Describe what the thing does and what it costs you.
- **Every command must have been run by the author**, on the machine and OS stated in the tutorial. If you adapted a command from documentation without running it, remove it.
- **No em dashes or en dashes.** Use a comma, a colon, or parentheses. This is enforced in review.
- **Every code block carries a language tag**: ` ```bash `, ` ```json `, ` ```jsonc `, ` ```text `. Untagged blocks fail review.
- **Screenshots need alt text** that describes the state being shown, not the file. "The MCP servers panel listing crmsolid with 13 tools" is alt text. "screenshot 3" is not. Prefer a text block over a screenshot when the content is text: screenshots go stale and are not searchable.
- **Second person, present tense, short sentences.** "You call the tool. It returns a count and an array." Not "one might then proceed to invoke".
- **Numbers over adjectives.** Say "12 platforms" or "returns in about 400 ms", not "many platforms" or "fast".
- **Say what does not work.** If a client cannot do the thing, write the limitation and the date you checked it.
- Tutorials are numbered files in `./tutorials/`. Cross link with relative paths (`./tutorials/02-ai-dm-triage-workflow.md`, `../README.md`).

## The review bar

A tutorial is merged when a reviewer who is not the author reproduces it from a clean machine. Clean means: a fresh user account or container, nothing preinstalled beyond the OS and the runtime the tutorial names, and no environment variables set by hand except the ones the tutorial names.

The reviewer checks:

- Every command runs and produces output close enough to what the page shows.
- Every verification step actually fails when the previous step is skipped. A verification that passes either way is not a verification.
- Nothing writes to a real audience without an explicit confirmation step first. Drafts, sandbox accounts and dry runs are the default in examples.
- No credential, key, token, cookie or account handle belonging to a real person appears in the text, a screenshot, or a JSON sample.
- Links resolve, including the relative ones.

If reproduction fails, the reviewer posts the exact command and the exact output. You fix it or you close the PR. There is no partial merge.

## Reporting an outdated client or server detail

The client table in [README.md](./README.md) goes stale faster than anything else. To correct it, open an issue titled `client: <name> <what changed>` and include:

- The row as it currently reads.
- The row as it should read.
- A link to the vendor's own documentation or release notes showing the change, plus the date you checked.
- The client version you observed it in.

Second hand reports without a source link are closed, because nobody can re verify a claim with no origin. The same format applies to server details: tool names, scopes and flags change between releases, so name the version.

## Licensing

By opening a pull request you agree that your prose, tables and diagrams are contributed under **CC BY 4.0**, and your code samples, configuration files and scripts are contributed under the **MIT license**. You confirm the work is yours to license, and that anything adapted from elsewhere is attributed inline with a link. Do not paste vendor documentation verbatim. Summarize it and link to the source.

## Checklist before opening a pull request

- [ ] Ran every command in the page, in order, on a clean machine.
- [ ] Every stage ends in a verification step that fails if the stage was skipped.
- [ ] Dash check is clean: `node -e "process.exit(/[\u2013\u2014]/.test(require('fs').readFileSync(process.argv[1],'utf8'))?1:0)" yourfile.md`
- [ ] Every code block has a language tag.
- [ ] Every image has alt text.
- [ ] No real keys, tokens, handles or customer names anywhere, including screenshots.
- [ ] Primary keyword appears in the H1, in the first 100 words, and in at least one H2. Nowhere else on purpose.
- [ ] Relative links to other tutorials resolve.
- [ ] Word count stated in the PR description.
- [ ] No marketing adjectives from the banned list.
