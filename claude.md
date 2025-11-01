# Claude Context

## Project Documentation

The complete design document for this project is located at:
- **design.md** - Contains the full architecture, user flows, data models, and technical specifications for the Scrappy Bot family photo backup system.

## Project Overview

Scrappy Bot is an SMS-based family photo/video backup bot built with C# .NET that:
- Receives photos/videos via SMS/MMS (Twilio)
- Uses conversational confirmation flows
- Backs up to Google Cloud Storage
- Organizes by date (YYYY/MM)
- Supports authorized users only (whitelist)

## Key Technical Details

- **Language:** C# / .NET 8+
- **Framework:** ASP.NET Core Web API
- **Messaging:** Twilio (SMS/MMS)
- **Storage:** Google Cloud Storage
- **State Management:** Redis
- **Authorization:** Phone number whitelist
- **Command Prefix:** @scrappy (required for all bot commands)
