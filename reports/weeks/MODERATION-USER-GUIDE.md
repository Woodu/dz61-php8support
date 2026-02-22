# Moderation Panel User Guide

**Version**: 1.0.0
**Last Updated**: 2026-02-18
**Audience**: Forum Moderators and Administrators

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Dashboard Overview](#dashboard-overview)
3. [Thread Moderation](#thread-moderation)
4. [Reply Moderation](#reply-moderation)
5. [User Management](#user-management)
6. [Viewing Logs](#viewing-logs)
7. [Permission Levels](#permission-levels)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Getting Started

### Access Requirements

To access the Moderation Panel, you must:

1. **Be logged in** to your forum account
2. **Have moderator status** (adminid = 1, 2, or 3)
3. **Be assigned** to moderate specific forums (for moderators)

**Access URL**: `https://yourforum.com/modcp.php`

### First Time Login

1. Navigate to `/modcp.php`
2. You'll be prompted to enter your password for verification
3. After verification, you'll see the moderation dashboard

**Note**: Super admins and super moderators have access to all forums. Regular moderators can only access forums they've been assigned to.

---

## Dashboard Overview

### Dashboard Features

The main dashboard (`/modcp.php`) provides:

- **Recent Activity**: Last 10 moderation actions across all moderators
- **Quick Navigation**: Fast access to all moderation features
- **User Information**: Your username, UID, and permission level

### Navigation Menu

- **管理首页** (`/modcp.php?action=home`) - Dashboard
- **主题管理** (`/modcp.php?action=moderate&op=threads`) - Moderate threads
- **回复管理** (`/modcp.php?action=moderate&op=replies`) - Moderate replies
- **用户管理** (`/modcp.php?action=members&op=ban`) - Ban/unban users
- **审核日志** (`/modcp.php?action=logs`) - View moderation logs
- **返回论坛** (`/`) - Back to forum index

---

## Thread Moderation

### Accessing Thread Moderation

1. Go to `/modcp.php?action=moderate&op=threads`
2. Select the forum you want to moderate (if multiple)
3. View threads awaiting moderation

### Understanding Thread Status

**Awaiting Moderation** (displayorder = -2):
- Threads that need moderator approval before appearing
- Created by users with posting restrictions
- Not visible to regular users

**Ignored Threads** (displayorder = -3):
- Threads previously marked as ignored
- Can be filtered in/out using the checkbox

### Moderating Threads

Each thread has three action options:

#### ✅ 通过 (Validate)

**When to use**: Thread meets forum guidelines and rules

**What happens**:
- Thread becomes visible to all users
- `displayorder` set to 0
- Thread marked as moderated
- Author may receive credits (if configured)
- **Action is logged** in moderation logs

**How to use**:
1. Check the "通过" checkbox for the thread
2. Click "通过" button
3. Thread is approved immediately

#### 🗑️ 删除 (Delete)

**When to use**: Thread violates rules or guidelines

**What happens**:
- **With recycle bin**: Thread moved to recycle bin (displayorder = -1)
- **Without recycle bin**: Thread permanently deleted
- All posts, attachments, polls, trades are deleted
- **Action is logged** in moderation logs

**How to use**:
1. Check the "删除" checkbox for the thread
2. Click "删除" button
3. Confirm deletion in popup

**Note**: Check if forum has recycle bin enabled before deleting.

#### ⏭️ 忽略 (Ignore)

**When to use**: Thread needs review but can wait

**What happens**:
- Thread marked as ignored (displayorder = -3)
- Not visible to regular users
- Can be reviewed later via filter toggle
- **Action is logged** in moderation logs

**How to use**:
1. Check the "忽略" checkbox for the thread
2. Click "忽略" button
3. Thread is marked as ignored

### Batch Operations

To perform actions on multiple threads at once:

1. Check the checkbox next to each thread you want to moderate
2. The batch bar appears at the top showing selected count
3. Choose action: "通过", "删除", or "忽略"
4. All selected threads are processed
5. Results show success/failure for each thread

**Tips**:
- Use batch operations to speed up moderation
- Review each thread carefully before batch operations
- Check for duplicate or spam posts
- Partial failures are handled gracefully

---

## Reply Moderation

### Accessing Reply Moderation

1. Go to `/modcp.php?action=moderate&op=replies`
2. Select the forum you want to moderate
3. View replies awaiting moderation

### Understanding Reply Status

**Awaiting Moderation** (invisible = -2):
- Replies that need moderator approval
- Posted by users with posting restrictions
- Not visible to regular users

**Ignored Replies** (invisible = -3):
- Replies previously marked as ignored
- Can be toggled via filter checkbox

### Moderating Replies

Each reply has three action options:

#### ✅ 通过 (Validate)

**When to use**: Reply meets forum guidelines

**What happens**:
- Reply becomes visible
- `invisible` set to 0
- Thread reply count updated
- Author may receive credits
- **Action is logged** in moderation logs

#### 🗑️ 删除 (Delete)

**When to use**: Reply violates rules

**What happens**:
- Reply permanently deleted from database
- Attachments cleaned up
- Thread reply count updated
- **Action is logged** in moderation logs

#### ⏭️ 忽略 (Ignore)

**When to use**: Reply needs review but can wait

**What happens**:
- Reply marked as ignored (invisible = -3)
- Can be reviewed later
- **Action is logged** in moderation logs

### Filter Options

**Toggle**: "只显示被忽略的回复"
- Shows only ignored replies (invisible = -3)
- Default: Shows awaiting moderation (invisible = -2)

### Batch Operations

Same as thread moderation - use checkboxes to select multiple replies and perform batch actions.

---

## User Management

### Banning Users

#### Access Ban Form

1. Go to `/modcp.php?action=members&op=ban&uid=123`
   - Replace 123 with the user ID you want to ban
   - Or access via user profile link

2. View user information:
   - Username
   - Current group
   - Previous bans (if any)

3. Configure ban:
   - **Duration**: Number of days (0 = permanent)
   - **Reason**: Why you're banning the user

4. Click "确认封禁" (Confirm Ban)

#### Ban Duration Guidelines

| Duration | Use Case |
|----------|----------|
| 1-3 days | Minor infractions, first offense |
| 7 days | Standard ban period |
| 14-30 days | Serious or repeat offenses |
| 0 (permanent) | Severe violations (spam, abuse) |

#### What Happens When Banning

- User's group changed to "Banned" (groupid = 4)
- Original group saved to `groupterms` field
- Expiry timestamp set
- User cannot login or post
- **Ban is logged** with your username and reason

### Unbanning Users

#### Access Unban Form

1. Go to `/modcp.php?action=members&op=ban&uid=123`
2. If user is banned, you'll see "解禁" (Unban) button
3. Click "解禁"
4. User's original group is restored

#### What Happens When Unbanning

- User's group restored from `groupterms['main']`
- Group expiry cleared
- `groupterms['main']` deleted
- User can login and post again
- **Unban is logged** in moderation logs

#### Unban Scenarios

**Automatic Restoration**: If `groupterms['main']` exists:
- Original groupid restored
- Original adminid restored
- Ban expiry cleared

**No Previous Group**: If `groupterms['main']` doesn't exist:
- Default member group assigned (groupid = 10)
- adminid set to 0 (regular member)

---

## Viewing Logs

### Accessing Moderation Logs

1. Go to `/modcp.php?action=logs`
2. View all moderation actions across the forum

**Permission Required**:
- Super admins (adminid = 1) and super moderators (adminid = 2)
- Regular moderators (adminid = 3) **cannot** view logs

### Log Information

Each log entry shows:

| Field | Description |
|-------|-------------|
| **Time** | When the action occurred |
| **Action** | What was done (delete_thread, ban_user, etc.) |
| **Moderator** | Who performed the action (username + UID) |
| **Target** | What was moderated (thread/post/user ID) |
| **Reason** | Why the action was taken (if provided) |

### Filter Options

**Filter by Action**:
- View specific moderation actions only
- Examples: `delete_thread`, `ban_user`, `pin_thread`

**Filter by Forum**:
- View actions in specific forum
- Useful for tracking forum-specific issues

**Filter by Moderator**:
- View actions by specific moderator
- Useful for auditing moderator activity

---

## Permission Levels

### Super Administrator (adminid = 1)

**Can Do**:
- Everything
- Ban anyone (including other admins)
- Set global pins and digests
- View all logs
- Access all forums

**Cannot**:
- Be banned by anyone

### Super Moderator (adminid = 2)

**Can Do**:
- Moderate all forums
- Ban regular users (groupid 10, 11, etc.)
- Set section-level pins and digests
- View all logs
- Most admin functions

**Cannot**:
- Ban super admin
- Ban other super moderators
- Set global pins (super admin only)

### Forum Moderator (adminid = 3)

**Can Do**:
- Moderate assigned forums only
- Validate, delete, ignore posts
- Close/open, pin/unpin, digest/undigest threads
- Move threads between assigned forums

**Cannot**:
- Moderate unassigned forums
- Ban users
- View logs
- Set section or global pins

### Regular Member (adminid = 0)

**Can Do**:
- View own content
- Edit own posts (if allowed)
- Report content to moderators

**Cannot**:
- Access moderation panel
- Moderate any content
- Ban users
- View logs

---

## Best Practices

### 1. Thread Moderation

**DO**:
- Read the full thread before deciding
- Check if thread is in the correct forum
- Consider moving thread instead of deleting
- Provide clear reasons when deleting or ignoring

**DON'T**:
- Delete threads without reviewing content
- Delete threads without moving to correct forum first
- Ignore threads that need to be deleted

### 2. Reply Moderation

**DO**:
- Review context by reading surrounding posts
- Check if reply is spam or low-quality
- Validate good replies quickly
- Delete rule violations

**DON'T**:
- Delete opinions you disagree with
- Ignore valid posts that happen to be critical
- Validate spam or low-effort posts

### 3. User Bans

**DO**:
- Start with shorter bans (7 days) for first offenses
- Provide clear, specific reasons
- Consider warnings for minor infractions
- Document ban reasons thoroughly

**DON'T'T**:
- Ban permanently on first offense (except spam bots)
- Ban without explanation
- Ban based on personal disagreements
- Ban users for expressing opinions

### 4. Using Logs

**DO**:
- Review logs periodically to catch patterns
- Check logs if you notice suspicious activity
- Use logs to identify problematic users
- Review other moderators' actions for consistency

**DON'T**:
- Use logs to spy on other moderators
- Criticize moderators based on logs
- Ignore patterns that suggest systematic issues

---

## Troubleshooting

### Common Issues

#### "Access Denied" Error

**Problem**: You see "Access denied" when trying to moderate

**Possible Causes**:
1. Not logged in
2. Not a moderator (adminid = 0)
3. Trying to moderate forum you're not assigned to
4. Trying to perform action above your permission level

**Solutions**:
1. Make sure you're logged in
2. Check your adminid level
3. Verify you're assigned to the forum
4. Check permission matrix for action requirements

#### "Thread Not Found" Error

**Problem**: Thread doesn't exist or already deleted

**Possible Causes**:
1. Thread was already deleted
2. Thread ID is incorrect
3. Thread was moved and you're looking in wrong forum

**Solutions**:
1. Refresh the page
2. Check thread ID from URL
3. Search for thread in forum

#### "Permission Denied" for User Actions

**Problem**: Can't ban or unban user

**Possible Causes**:
1. Not an admin (adminid = 3 is moderator only)
2. Trying to ban someone with higher/same adminid
3. User not actually banned (when trying to unban)

**Solutions**:
1. Check your adminid level
2. Verify target user's adminid
3. Check if user is currently banned

#### Batch Operations Fail

**Problem**: Some threads in batch operation failed

**Possible Causes**:
1. Threads already deleted
2. Threads in forums you can't moderate
3. Database errors

**Solutions**:
1. Review error messages for each failed item
2. Verify you have permission for all items
3. Try smaller batches
4. Check server logs for database errors

---

## Keyboard Shortcuts

### When Moderating Threads

- **Check All**: Not currently implemented
- **Invert Selection**: Not currently implemented
- **Refresh Page**: F5 or browser refresh button

### Future Enhancements

Planned features for future versions:

1. **AJAX Operations** - Faster feedback without page reload
2. **Bulk Select All** - Select all threads on page
3. **Advanced Filters** - Filter by date range, user, etc.
4. **Export Logs** - Download logs as CSV/Excel
5. **Statistics Dashboard** - Charts and graphs of moderation activity
6. **Real-time Updates** - WebSocket for live moderation queue

---

## Glossary

| Term | Definition |
|------|------------|
| **Recycle Bin** | Temporary holding area for deleted threads before permanent deletion |
| **Soft Delete** | Moving thread to recycle bin (displayorder = -1) |
| **Hard Delete** | Permanently deleting thread from database |
| **Display Order** | Thread sort order (0 = normal, -1 = deleted, -2 = awaiting moderation, -3 = ignored) |
| **Invisible** | Post visibility (0 = visible, -2 = awaiting, -3 = ignored) |
| **AdminID** | Permission level (1 = super admin, 2 = super mod, 3 = mod, 0 = member) |
| **GroupID** | User group (4 = banned, 10 = member, etc.) |
| **Groupterms** | Serialized data storing original group info for banned users |

---

## Additional Resources

### Internal Documentation
- [API Documentation](MODERATION-API.md) - Complete API reference
- [Week 12 Design Document](week12-design.md) - Technical design
- [Week 12 Completion Report](WEEK12-COMPLETE.md) - Project summary

### External Resources
- Discuz! 6.1F Original Documentation
- PHP 8.3 Migration Guide
- Forum Rules and Guidelines

### Getting Help

If you encounter issues not covered in this guide:

1. Check the [Troubleshooting](#troubleshooting) section
2. Review error messages carefully
3. Consult with other moderators
4. Contact forum administrators

---

**Last Updated**: 2026-02-18 22:00 UTC
**Version**: 1.0.0
**Next Review**: After next deployment
