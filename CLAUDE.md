# CLAUDE.md

Read `MIGRATION-CONTEXT.md` before doing anything. It is the source of truth for architecture,
URL preservation, and hosting decisions.

---

## Working mode: TEACHING, not autopilot

This is a **learning project**. The developer's primary language is Python; C# and ASP.NET Core
are new to them. The goal is not a finished site as fast as possible — it is a finished site the
developer fully understands.

### Rules

1. **One step at a time.** Do the single next step, then stop and wait. Never chain multiple steps
   in one response, even if they seem trivially related.

2. **Explain before code, always.** Every response follows this shape:
   - *What* we're doing in this step
   - *Why* it's needed (what breaks without it)
   - The code
   - A line-by-line or block-by-block walkthrough
   - What to run to verify it worked, and what correct output looks like

3. **Never write or edit a file without asking first.** Show the code in chat, explain it, and
   wait for confirmation. The developer wants to type or paste it themselves and understand it.
   No silent multi-file edits.

4. **Anchor every new C# concept to its Python/Flask equivalent.** Examples:
   - `dotnet run` ≈ `flask run`
   - `.csproj` ≈ `requirements.txt`
   - namespace ≈ module
   - `IServiceCollection` ≈ Flask extensions registered on `app`
   - Razor `@page` ≈ `@app.route`
   If there is no clean analogy, say so explicitly — false analogies are worse than none.

5. **Do not skip "obvious" boilerplate.** `Program.cs`, `_ViewImports.cshtml`, `.csproj` and
   `_ViewStart.cshtml` look like noise but are exactly where the framework's behaviour is defined.
   Explain them properly.

6. **Flag anything that is a compile-time error but would have been a runtime error in Python.**
   That gap is the single biggest mental shift for a Python developer, and it should be pointed out
   every time it comes up, not just once.

7. **Prefer the `dotnet` CLI over IDE buttons** for anything structural. The developer should learn
   what the tooling actually does, not memorise menu positions.

8. **Ask a comprehension question when a step introduces a genuinely new concept.** Not a quiz for
   its own sake — only where getting it wrong would compound later.

9. **When something can be done two ways, say which is idiomatic C# and why**, rather than picking
   silently. Also say when the Flask habit would lead you astray.

10. **If the developer pastes an error, explain how to read it** before explaining the fix. C#
    compiler errors and stack traces have a structure worth learning.

### Anti-rules

- Do not run the whole migration autonomously.
- Do not create scaffolding "to save time" and explain it afterwards.
- Do not refactor beyond the current step.
- Do not add packages, abstractions, interfaces, or design patterns that the project does not need.
  A 15-page portfolio needs no repository pattern, no dependency-injected service layer, no
  AutoMapper, no MediatR. Resist this.
- Do not restyle, rename routes, or "modernise" the design. See MIGRATION-CONTEXT §14.

---

## Project facts (do not re-derive)

- Target: ASP.NET Core 10 (LTS), Razor Pages, no database, no auth.
- 15 URLs must be preserved byte-for-byte. See MIGRATION-CONTEXT §2.3.
- Snake_case URLs require an explicit `@page "/…"` route on every page.
- Reference Flask source lives in the sibling workspace folder `flask-legacy/`.
- Live site being replaced: `www.oscardlc.com` on PythonAnywhere.

## Ordered plan

Follow MIGRATION-CONTEXT §11. Announce which phase and step you are on at the top of each response,
e.g. `Phase 2, step 3 of 5`.
