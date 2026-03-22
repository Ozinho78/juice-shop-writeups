# Challenge #3 – OSINT Chain

> **Category:** Sensitive Data Exposure  
> **Difficulty:** ⭐⭐⭐
> **Tools:** ExifTool, Burp Suite Repeater, SecLists  
> **Video:** ▶ [Watch on Loom](https://loom.com/TBD)

---

## Table of Contents

- [Overview](#overview)
- [Vulnerability Description](#vulnerability-description)
- [Challenge 3.1 – Geo-Stalking](#challenge-31--geo-stalking)
  - [Reconnaissance](#reconnaissance)
  - [Walkthrough – Metadata Extraction with ExifTool](#walkthrough--metadata-extraction-with-exiftool)
- [Challenge 3.2 – Bjoern's Favourite Pet](#challenge-32--bjoerns-favourite-pet)
  - [Reconnaissance](#reconnaissance-1)
  - [Walkthrough – Security Question Bypass](#walkthrough--security-question-bypass)
- [References](#references)

---

## Overview

This writeup covers an OSINT-based attack chain against the OWASP Juice Shop that demonstrates how publicly available metadata and social engineering can be combined to perform a full account takeover — without any technical exploitation of the application itself.

The chain consists of two linked challenges:

1. **Geo-Stalking** — extract GPS coordinates from an image uploaded to the Juice Shop Photo Wall to determine a user's location
2. **John's Favourite Place to Hike** — use OSINT research to answer Johns's security question and reset his account password

Both challenges are performed against a clean local Docker instance on `http://localhost:4000`.

---

## Vulnerability Description

**Sensitive Data Exposure via Metadata** — Image files captured by smartphones and cameras embed rich metadata in the EXIF format, including GPS coordinates, device model, timestamp, and camera settings. When users upload photos to a web application without server-side metadata stripping, this information becomes publicly accessible to anyone who can download the file.

**Weak Security Questions** — Security questions represent a fundamentally flawed authentication recovery mechanism. Answers to questions like "What is your pet's name?" are often discoverable through social media, public profiles, or simple OSINT research — effectively making the account recovery process less secure than the original password.

---

## Challenge 3.1 – Geo-Stalking

### Reconnaissance

The Juice Shop Photo Wall (`/#/photo-wall`) displays images uploaded by users. These images are served directly from the server without metadata stripping, meaning any EXIF data embedded at upload time is still present in the downloadable file.

The target image was uploaded by a user whose location we want to determine.

### Walkthrough – Metadata Extraction with ExifTool

**Step 1 – Navigate to the Photo Wall**

Open the Juice Shop and navigate to:
```
http://localhost:4000/#/photo-wall
```

> [!NOTE]
> The Photo Wall link may not appear in the main navigation. If so, navigate directly via the URL above or find the route via `main.js` inspection as shown in Challenge #2.

**Step 2 – Identify the target image**

Browse the Photo Wall and identify an image uploaded by the target user. The challenge hints at a specific image — look for one with a caption referencing a memorable or scenic location. In this case we are looking for John's photo. We already got his email address from the admin section in one of the previous challenges.

**Step 3 – Download the image**

Right-click the image → **Save image as** → save locally, e.g. `favorite-hiking-place.png`

Alternatively, intercept the image request in Burp Suite to get the direct URL, then download via:

```
http://localhost:4000/assets/public/images/uploads/<filename>.jpg
```

**Step 4 – Extract EXIF metadata with ExifTool**

**Windows:**
```powershell
exiftool.exe favorite-hiking-place.png
```

**Linux / macOS:**
```bash
exiftool favorite-hiking-place.png
```

The output contains the embedded GPS coordinates:

```
GPS Latitude  : 36 deg 57' 31.38" N
GPS Longitude : 84 deg 20' 55.04" W
GPS Position  : 36 deg 57' 31.38" N, 84 deg 20' 55.04" W
```

**Step 5 – Look up the coordinates**

Copy the GPS coordinates and paste them into Google Maps:
```
36°57'31.38"N 84°20'55.04"W
```

The coordinates resolve to a specific location — this is the answer required to solve the Geo-Stalking challenge in the Juice Shop.

**Step 6 – Submit the location in Juice Shop**

The Juice Shop challenge asks you to identify the location. Navigate to the Score Board and trigger the challenge validation by submitting the correct answer when prompted.

---

## Challenge 3.2 – Bjoern's Favourite Pet

### Reconnaissance

Bjoern Kimminich is the creator of the OWASP Juice Shop. His account (`bjoern@owasp.org`) uses a security question — **"What is the name of your favourite pet?"** — as the password reset mechanism.

The answer is publicly discoverable through OSINT:

1. Search for Bjoern Kimminich on social media (Twitter/X, LinkedIn, GitHub)
2. Look for posts, photos, or mentions related to pets
3. The pet's name **Zaya** can be found in his public social media posts

This demonstrates that security questions based on personal information are trivially bypassable for anyone with a public online presence.

### Walkthrough – Security Question Bypass

**Step 1 – Navigate to the Forgot Password page**

```
http://localhost:4000/#/forgot-password
```

**Step 2 – Enter Bjoern's email address**

```
bjoern@owasp.org
```

The page displays his security question:
```
What is the name of your favourite pet?
```

**Step 3 – Enter the OSINT-discovered answer**

```
Zaya
```

**Step 4 – Set a new password**

Enter any new password — the application accepts it without any additional verification. Bjoern's account password has now been reset.

**Step 5 – Log in as Bjoern**

Navigate to the login page and log in with:
```
Email:    bjoern@owasp.org
Password: <your newly set password>
```

The Juice Shop awards the **"Bjoern's Favourite Pet"** challenge upon successful login.

---

## References

- [OWASP Top 10 – A02:2021 Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
- [ExifTool – Official Documentation](https://exiftool.org/)
- [OWASP – Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [OWASP – File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)

---
