# Scrappy Bot

An SMS-based bot that automatically backs up family photos and videos to Google Cloud Storage. Built for parents who want to preserve precious memories shared via text messages without the friction of manual uploads.

## What It Does

Scrappy Bot sits in your family group chat and watches for photos and videos. When you share a photo:

1. The bot privately messages you with the photo
2. You confirm whether to save it by replying `@scrappy yes` or `@scrappy no`
3. Approved photos are automatically backed up to Google Cloud Storage, organized by date
4. No apps to install, no manual uploads - just text as usual

## Key Features

- **Zero Friction**: Works entirely via SMS/MMS - no new apps needed
- **Group Sharing**: Share photos in the family group chat, confirm privately
- **Always Shows Media**: You always see exactly what you're confirming
- **One at a Time**: Handles photo bursts gracefully with queueing
- **Video Support**: Backs up videos with thumbnail-based confirmation
- **Secure**: Whitelist-based authorization - only approved family members can use it
- **Date-Organized**: Automatic organization by year/month (e.g., 2025/10/)
- **Timeout Handling**: Graceful cleanup of unconfirmed photos after 48 hours

## Technology Stack

- **Backend**: C# / .NET 8+ (ASP.NET Core Web API)
- **Messaging**: Twilio (SMS/MMS)
- **Storage**: Google Cloud Storage
- **State Management**: Redis
- **Deployment**: Docker on VPS

## Configuration

### Environment Variables

Create a `.env` file in the project root with your Twilio credentials:

\`\`\`
SMS_ACCOUNT_SID=your_twilio_account_sid
SMS_AUTH_TOKEN=your_twilio_auth_token
\`\`\`

See `.env.example` for the template.

## Documentation

See `design.md` for the complete system architecture, workflows, and technical specifications.

## Cost

Estimated monthly operating cost for 2 users with moderate usage: ~$23/month (~$273/year)

- VPS Hosting: $10/month
- Twilio (phone number + messages): ~$12/month
- Google Cloud Storage: ~$1/month

## Getting Started

(Setup instructions will be added as the project develops)

## License

See `LICENSE` file for details.
