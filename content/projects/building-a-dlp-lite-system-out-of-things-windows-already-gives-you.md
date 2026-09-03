+++
title = "Building a DLP-lite system out of things Windows already gives you"
date = 2026-09-03T21:03:31+02:00
description = "Building a DLP-lite with PowerShell, NTFS alternate data streams and Common File Dialog hooks. Architecture, trade-offs and the limits of the approach"

[taxonomies]
tags = ["blue-team", "windows", "active-directory", "dlp", "powershell", "csharp"]
+++

## The constraint that shaped everything

A while back I found myself with a familiar small-IT problem: no budget for an enterprise DLP suite, no appetite from the leadership to open a new hosted service and a genuine need to stop the *accidental* leak, the contract that gets dragged into the wrong upload box, not the determined insider.

The brief I set for myself was deliberatly narrow:

- Use only what's already licensed: Active Directory, Group Policy, PowerShell, RSAT.
- No new servers, no new SaaS, no new attack surface.
- Reduce *accidental* exposure. Not an enterprise DLP replacement and not pretending to be one.

This post is a walkthrough of the architecture that came out of that constraint, the parts that worked, the parts that were harder than expected and the structural limits of uilding security tooling this way in general.

## Architecture in one picture

Two independent-but-coordinated pieces, talking to a polixy file and a log folder on an existing file share:

![Architecture of the dlp-lite system: a PowerShell crawler on the server side and a C# agent service with three client-side hooks, both reading a shared policy file on an existing network share](/images/projects/dlp-lite/dlp-architecture.svg)

Nothing here needs new infrastructure. The crawler runs as a Scheduled Task against a file share that already exists. The agent runs as a Windows Service, Everything else hooks into things Windows already exposes.

## Design decision #1: classification has to live *inside* the file

The first version of this idea used a sidecar file, `report.pdf` next to `report.pdf.classification`. It's the obvious approach and it's wrong: the moment a user copies or moves just the original file (which is the normal, overwhelmingly common thing to do), the sidecar gets left behind silently. A classification mechanism that depends on the user's voluntary cooperation to carry a second file along is broken before it starts.

So classification has to be written **inside** the file, using whatever mechanism the format already gives you:

- **Office formats** (`.docx`, `.xlsx`, `.pptx`): these are zip archives; write a custom document property into `docProps/custom.xml`.
- **PDF**: write into the document's Info dictionary, the same place title/author metadata lives. Survives copying regardless of destination.
- **Images** (`.jpg`, `.tiff`): a custom EXIF tag.
- **Everything else** (CSV, TXT, ZIP, arbitrary binaries): an NTFS **Alterante Data Stream** attached to the file itself. It's invisibile in Explorer, doesn't touch the content the user sees and critically travels with the file automatically when copied with standard Windows tools (`Copy-Item`, Explorer copy/paste), because it's not a separate file, it's an extra stream *on* the same file.

The filename tag (`[RIS] contract.pdf`) stays too, as a *visible* signal, think of it as the belt, with the embedded metadata as the suspenders. If the two disagree, the more restrictive one always wins. You never trust the more permissive signal, because that would mean one edit is enough to bypass the whole thing.

```
function Set-NTFSClassificationStream {
    param([string]$FilePath, [string]$TagValue)
    try {
        Set-Content -Path $FilePath -Stream "classification" -Value $TagValue -Force
        return $true
    }
    catch {
        return $false
    }
}
```

Reading it back is the mirror image, same `:streamname` syntax:

```
$streamPath = "$path:classification"
Get-Content -Path $streamPath
```

## Design decision #2: one brain, four mouths

The hard part isn't detecting a classified file, that's a few lines of PowerShell. The hard part is that "a user is about to upload a file" can happen from at least four different surfaces, each with a different API surface and a different blind spot:

- A **browser** sees only what's inside the page, a `File` object with a name and byte content, but by design, **never a full filesystem path**. That's a deliberate browser security boundary, not an oversight to work around.
- **Outlook**'s "Attach file" dialog isn't Outlook UI at all, it's the standard Windows Common File Dialog, the same one Explorer and most Win32 apps use.
- Drag-and-drop onto a web page and pasting a file directly, bypass both of the above.
- Outlook's send action is a separate moment entirely and the last chance to catch a file that was attached *before* it was classified.

Rather than writing a bespoke integration for every single app that might open a file picker, the design leans on the one thing almost every desktop program shares: the **Common File Dialog**. Windows exposes an accessibility API (originally meant for screen readers) that lets a process observe, system-wide, when one of these dialogs appears and read what's seleceted in it, no injection into the target process required.

That covers Explorer, Outlook, FTP clients and most Win32 software with a single comment. What it *can't* see is drag-and-drop directly onto a web page (that never opens a system dialog) or paste, so a lightweight browser extension covers exactly that gap and only that gap. The Outlook send-time hook covers a third, narrower case: a file attached before classification or dropped straight onto the compose window.

All four components ask the same question to the same place, a small Windows Service exposing its classification logic over a local named pipe, instead of reimplementing "what does \[RIS\] mean" four times in four languages:

```
private string ClassifyAndDecide(string request)
{
    string classification;

    if (request.StartsWith("NAME|"))
    {
        // Browser drag&drop/paste only ever gives us a filename,
        // never a path, classification here can only be name-based.
        var fileName = request.Split('|').ElementAtOrDefault(1) ?? "";
        classification = ClassifyFromName(filename);
    }
    else if (File.Exists(requet))
    {
        classification = ClassifyFromPath(request); // name + embedded metadata + NTFS stream
    }
    else
    {
        classification = "NONE";
    }

    bool blocked = classification == "RESTRICTED" || classification == "INTERNAL";
    return blocked ? $"BLOCK|{classification}" : $"{ALLOW|classification}";
}
```

## What was harder than it looked

**Digitally signed PDFs and contracts.** Rewritting a PDF's Info dictionary or an Office file's zip archive, can invalidate an existing digital signature. For anything already signed, writing the NTFS stream only (which doesn't touch file content at all) is the safer path; native-metadata writing gets skipped for those cases.

**Session isolation.** A component that needs to see the user's desktop session (to hook dialogs, to show notifications) can't run as a SYSTEM service, Windows has isolated services for the interactive desktop since Vista. So the dialog-hook piece has to run in the user's own session, started at logon via a scheduled task, which in turn means it *can* be killed by a standard user from Task Manager, someting a real SYSTEM service can't be. Auto-restart on termination closes most of the gap, not all of it. This is a real, acknowledged trade-off of the approach, not a bug to eventually fix, a component with that much visibility into the user's session, running with elevated resistance to termination, starts trading away exactly the kind of restraint the whole "zero new attack surface" premise was built on.

**Rollout, not launch.** None of this went live in one step. The order that actually avoided breaking things:

1. Pilot: one folder, a couple of test machines, watch for anything that assumes fixed filenames.
2. Crawler running for a couple weeks, logging only.
3. Agent in shadow mode, logs what it *would* block, blocks nothing, for several weeks, specifically to surface false positives before they become support tickets or, worse, quiet loss of trust in the system.
4. Real blocking, first on the pilot group, then gradually elsewhere.

## Where this kind of approach stops being enough

This is the part worth being explicit about, because it's the part easiest to gloss over once something is working: a system like this is a **first-level control against accidental exposure**, not a defense against someone who wants to move data out on purpose.

A few structural reasons why, true of this pattern in general, not just one implementation:

- **Filename and metadata can be changed by anyone with write access to the file.** That's seconds of work, no special tooling. The system catches the person who didn't realize what they were sending, not the person who did and doesn't want to be caught.
- **Classification signals only survive as long as the file stays inside the channels the system watches.** An NTFS stream doesn't survive leaving an NTFS volume, email attachment, upload to an external service, a FAT32 USB stick. Native embedded metadata (PDF, Office, EXIF) is more durable, which is exactly why both are written for the formats that support it.
- **Device or application allow-listing controls *what* can run or connect, not *what* it does with the access it's given.** A whitelisted USB device or an approved FTP client for a specific vendor relationship is, by definition, a channel the system doesn't inspect the contents of.
- **Anything that doesn't touch a file on disk or a whitelisted app is invisible to this whole approach by construction.** Printing, a phone camera pointed at a screen, a screenshot. No file-classification system built this way sees those and it's worth saying that plainly rather than letting a "DLP" label imply otherwise.

None of that is an argument against building it, a first-level control that's honest about its limits, build for free out of things already licensed, is a reasonable thing to ship when the alternative is nothing. It *is* an argument for writing the limits down and saying them out loud to whoever owns compliance and risk, before go-live, not after an incident, so a partial control never gets mistaken, in an audit or a postmortem, for protection that we never actually there.

---

A reference-implementation PoC of this architecture, crawler, agent service, session monitor, browser extension and Outlook add-in, wired against a fictional lab domain, is up on [github.com/GreyCipher-sec/dlp-lite](https://github.com/GreyCipher-sec/dlp-lite).
