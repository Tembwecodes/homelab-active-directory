# Step 7 – Automatic Department Drive Mapping with Group Policy

After setting up the file server in the previous step (the Finance, HR and IT shares on `\\DC01`,
each locked down with its own NTFS permissions), the obvious next job was to make those shares
actually turn up for people automatically. Nobody in a real company manually types `\\server\share`
every morning. The right drive just appears when they log in. So that's what I wanted to build: log
in as a Finance user and a Finance drive shows up, log in as HR and you get the HR drive instead, and
crucially you never see a department that isn't yours.

This is a really common task in a Windows environment, and it comes up in interviews a lot, because
it pulls together three things at once: file shares, security groups, and Group Policy.

## What I ended up with

One Group Policy Object, `Map-Department-Drives`, linked at the top of the domain, using **Group
Policy Preferences → Drive Maps** with **item-level targeting** to decide who gets what.

| Drive | Points to | Only for members of |
|-------|-----------|---------------------|
| F:    | `\\DC01\Finance` | Finance |
| H:    | `\\DC01\HR`      | HR |
| I:    | `\\DC01\IT`      | IT-Support |

The clever bit is that the "who gets this" condition lives on each individual drive, so a single GPO
sat at the domain level can still hand out three different drives to three different sets of people.
Everyone else just quietly skips the drives that aren't meant for them.

## The thing that made this click for me: Policies vs Preferences

Group Policy has two halves, and I didn't really appreciate the difference until I needed it here:

- **Policies** are enforced and constantly re-applied, so the user can't get around them. That's what
  I used earlier for the password policy.
- **Preferences** are applied too, but they're more flexible, and they support **item-level
  targeting**, which is a per-item rule like *"only do this if the user is in the Finance group."*

Drive Maps sit under Preferences, and that targeting feature is the whole reason this works without
me having to write any login scripts.

## Why I did this from the client, not the server

DC01 runs Server Core, so there's no GUI on it, and honestly that's how I want it, because it's
closer to how things are actually run. So instead of managing Group Policy on the server itself, I
did it remotely from the Windows 11 client using the Group Policy Management Console (part of RSAT),
which is exactly what an admin would do from their own workstation.

## How I built it

1. Installed the Group Policy Management Console (RSAT) on the Windows 11 client. This turned out to
   be the hard part, see below.
2. Created the `Map-Department-Drives` GPO in GPMC and linked it at the domain root.
3. Edited it and went to `User Configuration → Preferences → Windows Settings → Drive Maps`.
4. For each department, added a new mapped drive:
   - Action: `Create`
   - Location: the share path, e.g. `\\DC01\Finance`
   - Reconnect: ticked
   - A drive letter (F:, H:, I:)
   - Then on the **Common** tab: ticked **Item-level targeting**, opened **Targeting**, and added a
     **Security Group** condition pointing at the matching group.
5. Ran `gpupdate /force` on the client to pull the new policy straight away rather than waiting.
6. Tested it by logging in as actual department users.

## Did it work? Yes, and I proved it both ways

I tested with two different accounts on the same machine:

- Logged in as **dclark** (in the Finance group) and got **F: only**. No H:, no I:.
- Logged in as **ewilson** (in the HR group) and got **H: only**. No F:, no I:.

Same policy, same computer, two people, two completely different results. That's the bit that
actually proves item-level targeting is doing its job. It's filtering by group membership, not just
sticking a drive on for everyone. Both drives showed real free space too, so they were properly
connected to the shares, not just sitting there as dead icons.

Handy tip I picked up: in GPMC, if you select a drive-map item, the Processing pane shows
**"Filtered directly: Yes"**, which is a quick way to confirm targeting is set on it without opening
the whole thing up.

## Where the real learning was: the troubleshooting

The build itself is only a handful of clicks. Everything I actually learned came from things going
wrong, and working each one out by reading the error rather than guessing.

**The Group Policy console wasn't even installed.** `gpmc.msc` just did nothing. It turns out RSAT
isn't one single thing. It's a bundle of separate features, and I'd only ever installed the Active
Directory ones. The Group Policy console is its own component.

**Installing it failed with "Access is denied".** The first attempt hung on the progress bar for a
good ten minutes and then fell over with a DISM error. That interrupted run had left the component
store in a bad state, which was blocking the next attempt. Repairing it sorted it:

```powershell
DISM /Online /Cleanup-Image /RestoreHealth
```

Once that finished, the install
(`Add-WindowsCapability -Online -Name Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0`) went straight
through.

**Then the console couldn't reach the domain**, with the error "the specified domain either does not
exist or could not be contacted." After a moment of suspecting DNS, the actual reason was much
simpler: **DC01 was switched off.** With the domain controller down, the client could still log in on
cached credentials, but anything that needs live Active Directory (the Group Policy console, drive
maps, group lookups) had nothing to talk to. Once I started DC01 and gave it a minute or two to bring
up AD, DNS and SYSVOL, I confirmed it was ready:

```powershell
Test-Connection 192.168.1.10 -Count 2   # replies = the DC is reachable
nslookup homelab.local                  # resolves to 192.168.1.10 = DNS is serving the domain
```

(The first `nslookup` timed out once and then resolved, which is just what you get from a DC that's
only just finished booting.)

**A drive letter clash and a broken path.** When I first created the three drives, the HR one had
accidentally grabbed drive letter F: (same as Finance), and the IT one's path had saved as just
`\\DC01` with no share on the end. I caught both by actually reading back the list before testing
rather than after, and fixed them to H:/`\\DC01\HR` and I:/`\\DC01\IT`. Lesson learned: a single
wrong character in a path or a duplicate drive letter will just fail silently at logon, so it pays to
check the list first.

**And a few PowerShell typos along the way**, like `Test.Connection` instead of `Test-Connection`,
and `-COount` instead of `-Count`. Every one of them I spotted from the exact error PowerShell threw
back, which is genuinely a faster way to fix them than retyping and hoping.

## What this shows I can do

- Work with Group Policy Preferences and understand the Policies-vs-Preferences distinction
- Use item-level targeting to apply settings by security group
- Manage Group Policy remotely from a workstation (since Server Core has no GUI)
- Install optional Windows features with `Add-WindowsCapability`
- Repair a broken component store with DISM
- Troubleshoot domain connectivity and DNS (`Test-Connection`, `nslookup`, `gpupdate /force`)
- Check an AD account's state with `Get-ADUser` (whether it's enabled, what groups it's in)
- Work through problems methodically by reading the error, not guessing

## Why it matters

This is exactly how a real organisation gives the finance team their finance drive and the HR team
their HR drive from one central policy. It scales nicely too, since adding a new department is just a
new share, a new group, and one more targeted drive, rather than touching every machine by hand.
