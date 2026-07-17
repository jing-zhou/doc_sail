# SMTP Configuration Guide

## Overview
SMTP credentials are now stored in `gradle.properties` (local file, not committed to version control) and injected into `application.properties` at build time.

## Setup Instructions

### 1. Local Development
The `gradle.properties` file has been created with the current SMTP settings. This file is already added to `.gitignore` to prevent committing credentials.

**File: `gradle.properties`**
```properties
smtp.host=smtp-mail.outlook.com
smtp.port=587
smtp.username=your-email@example.com
smtp.password=your-password-here
smtp.auth=true
smtp.starttls.enable=true
smtp.starttls.required=true
```

### 2. Team Members / New Developers
If you're setting up the project for the first time:

1. Create a `gradle.properties` file in the project root directory
2. Copy the SMTP settings from the template above (or get credentials from your team lead)
3. Update with your SMTP credentials if needed

### 3. Production Deployment
For production, you have two options:

#### Option A: User-level gradle.properties
Create `~/.gradle/gradle.properties` in your home directory with the SMTP settings. This keeps credentials separate from the project.

#### Option B: Environment Variables
Set environment variables and reference them in `gradle.properties`:
```properties
smtp.host=${SMTP_HOST}
smtp.port=${SMTP_PORT}
smtp.username=${SMTP_USERNAME}
smtp.password=${SMTP_PASSWORD}
```

## How It Works

1. **Build Time**: The Gradle `processResources` task reads SMTP properties from `gradle.properties`
2. **Expansion**: Placeholders in `src/main/resources/application.properties` are replaced with actual values
3. **Runtime**: Spring Boot reads the expanded `application.properties` from the JAR file

## Security Notes

- ✅ `gradle.properties` is in `.gitignore` - credentials won't be committed
- ✅ Source `application.properties` contains only placeholders
- ✅ Built JAR contains actual credentials (secure the JAR appropriately)
- ⚠️ Never commit `gradle.properties` to version control
- ⚠️ Rotate credentials if accidentally exposed

## Verification

After building, verify the properties were injected correctly:
```bash
./gradlew clean processResources
cat build/resources/main/application.properties | grep "spring.mail"
```

You should see the actual SMTP values, not placeholders.

