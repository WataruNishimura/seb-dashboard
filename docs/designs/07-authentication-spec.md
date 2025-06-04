# Authentication System Design Specification

## 1. Overview

### 1.1 Purpose
The Authentication System provides secure, flexible authentication options allowing users to sign in using Email/Password or Single Sign-On (SSO), with the ability to link multiple authentication methods to a single account.

### 1.2 Tech Stack
- **Authentication Provider**: Auth0
- **Session Management**: JWT with secure cookies
- **Password Hashing**: bcrypt (via Auth0)
- **SSO Protocols**: SAML 2.0, OAuth 2.0, OpenID Connect
- **MFA**: TOTP (Time-based One-Time Password)

### 1.3 Key Features
- Email/Password authentication
- SSO integration (Google, Microsoft, SAML)
- Account linking (multiple auth methods per user)
- Optional MFA
- Password reset flow
- Email verification
- Session management
- Remember me functionality

## 2. Data Models

### 2.1 Extended User Model
```typescript
model User {
  id                String   @id // Primary identifier
  email             String   @unique
  emailVerified     Boolean  @default(false)
  emailVerifiedAt   DateTime?
  name              String
  nameJa            String?
  roles             Role[]
  locale            Language @default(JA)
  timezone          String   @default("Asia/Tokyo")
  isActive          Boolean  @default(true)
  isMfaEnabled      Boolean  @default(false)
  lastLoginAt       DateTime?
  lastLoginIp       String?
  lastLoginMethod   AuthMethod?
  passwordChangedAt DateTime?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  // Relations
  authIdentities    AuthIdentity[]
  sessions          UserSession[]
  loginHistory      LoginHistory[]
  passwordResets    PasswordReset[]
  
  @@map("users")
}

model AuthIdentity {
  id                String   @id @default(cuid())
  userId            String
  provider          AuthProvider
  providerUserId    String   // ID from the auth provider
  email             String?  // Email from this provider
  displayName       String?
  profileData       Json?    // Additional profile data
  isVerified        Boolean  @default(false)
  isPrimary         Boolean  @default(false)
  lastUsedAt        DateTime?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  user              User     @relation(fields: [userId], references: [id])
  
  @@unique([provider, providerUserId])
  @@index([userId, provider])
  @@map("auth_identities")
}

enum AuthProvider {
  EMAIL           // Email/Password
  GOOGLE          // Google OAuth
  MICROSOFT       // Microsoft OAuth
  SAML            // SAML SSO
  GITHUB          // GitHub OAuth
  
  @@map("auth_providers")
}

enum AuthMethod {
  PASSWORD
  GOOGLE_SSO
  MICROSOFT_SSO
  SAML_SSO
  GITHUB_SSO
  MAGIC_LINK
  
  @@map("auth_methods")
}
```

### 2.2 Session Management
```typescript
model UserSession {
  id                String   @id @default(cuid())
  userId            String
  sessionToken      String   @unique
  refreshToken      String?  @unique
  authMethod        AuthMethod
  ipAddress         String?
  userAgent         String?
  deviceInfo        Json?    // Device fingerprint
  isActive          Boolean  @default(true)
  lastActivityAt    DateTime @default(now())
  expiresAt         DateTime
  createdAt         DateTime @default(now())
  
  user              User     @relation(fields: [userId], references: [id])
  
  @@index([userId, isActive])
  @@index([sessionToken])
  @@index([expiresAt])
  @@map("user_sessions")
}

model LoginHistory {
  id                String   @id @default(cuid())
  userId            String?  // Null if login failed
  email             String   // Attempted email
  authMethod        AuthMethod
  provider          AuthProvider?
  success           Boolean
  ipAddress         String?
  userAgent         String?
  location          String?  // Geo-location from IP
  errorReason       String?  // If failed
  attemptedAt       DateTime @default(now())
  
  user              User?    @relation(fields: [userId], references: [id])
  
  @@index([email, attemptedAt])
  @@index([userId, attemptedAt])
  @@map("login_history")
}

model PasswordReset {
  id                String   @id @default(cuid())
  userId            String
  token             String   @unique
  hashedToken       String   // For secure comparison
  isUsed            Boolean  @default(false)
  usedAt            DateTime?
  expiresAt         DateTime
  requestedIp       String?
  createdAt         DateTime @default(now())
  
  user              User     @relation(fields: [userId], references: [id])
  
  @@index([token])
  @@index([userId, createdAt])
  @@map("password_resets")
}
```

### 2.3 MFA Models
```typescript
model MfaDevice {
  id                String   @id @default(cuid())
  userId            String
  deviceName        String
  type              MfaType
  secret            String   // Encrypted
  backupCodes       String[] // Encrypted backup codes
  isActive          Boolean  @default(true)
  isPrimary         Boolean  @default(false)
  lastUsedAt        DateTime?
  verifiedAt        DateTime
  createdAt         DateTime @default(now())
  
  user              User     @relation(fields: [userId], references: [id])
  
  @@index([userId, isActive])
  @@map("mfa_devices")
}

enum MfaType {
  TOTP            // Time-based OTP (Google Authenticator)
  SMS             // SMS OTP
  EMAIL           // Email OTP
  BACKUP_CODE     // One-time backup codes
  
  @@map("mfa_types")
}

model MfaChallenge {
  id                String   @id @default(cuid())
  userId            String
  sessionId         String?
  challengeType     MfaType
  code              String?  // Hashed
  isVerified        Boolean  @default(false)
  attempts          Int      @default(0)
  maxAttempts       Int      @default(3)
  expiresAt         DateTime
  verifiedAt        DateTime?
  createdAt         DateTime @default(now())
  
  @@index([userId, createdAt])
  @@map("mfa_challenges")
}
```

## 3. Authentication Flows

### 3.1 Email/Password Registration
```typescript
interface RegistrationFlow {
  // Step 1: User submits registration form
  async register(data: {
    email: string
    password: string
    name: string
    nameJa?: string
    locale: Language
  }): Promise<{ userId: string; verificationToken: string }>

  // Step 2: Create user in Auth0
  async createAuth0User(email: string, password: string): Promise<string>

  // Step 3: Create user in database
  async createDatabaseUser(auth0Id: string, userData: any): Promise<User>

  // Step 4: Send verification email
  async sendVerificationEmail(email: string, token: string): Promise<void>

  // Step 5: Verify email
  async verifyEmail(token: string): Promise<void>
}

// Implementation
class EmailPasswordRegistration {
  async register(data: RegistrationData) {
    // Validate input
    await this.validateRegistrationData(data)
    
    // Check if email already exists
    const existingUser = await prisma.user.findUnique({
      where: { email: data.email }
    })
    
    if (existingUser) {
      throw new ConflictError('Email already registered')
    }
    
    // Create user in Auth0
    const auth0User = await auth0.createUser({
      email: data.email,
      password: data.password,
      email_verified: false,
      user_metadata: {
        name: data.name,
        nameJa: data.nameJa,
        locale: data.locale
      }
    })
    
    // Create user in database
    const user = await prisma.user.create({
      data: {
        id: auth0User.user_id,
        email: data.email,
        name: data.name,
        nameJa: data.nameJa,
        locale: data.locale,
        authIdentities: {
          create: {
            provider: 'EMAIL',
            providerUserId: auth0User.user_id,
            email: data.email,
            isPrimary: true
          }
        }
      }
    })
    
    // Send verification email
    const verificationToken = await this.generateVerificationToken(user.id)
    await this.sendVerificationEmail(data.email, verificationToken)
    
    return { userId: user.id, message: 'Verification email sent' }
  }

  private validateRegistrationData(data: RegistrationData) {
    // Email validation
    if (!isValidEmail(data.email)) {
      throw new ValidationError('Invalid email format')
    }
    
    // Password strength validation
    const passwordErrors = validatePasswordStrength(data.password)
    if (passwordErrors.length > 0) {
      throw new ValidationError('Password does not meet requirements', passwordErrors)
    }
    
    // Name validation
    if (data.name.length < 2 || data.name.length > 100) {
      throw new ValidationError('Name must be between 2 and 100 characters')
    }
  }
}
```

### 3.2 Email/Password Login
```typescript
class EmailPasswordLogin {
  async login(credentials: {
    email: string
    password: string
    rememberMe?: boolean
  }): Promise<LoginResponse> {
    try {
      // Rate limiting check
      await this.checkRateLimit(credentials.email)
      
      // Authenticate with Auth0
      const auth0Response = await auth0.authenticate({
        grant_type: 'password',
        username: credentials.email,
        password: credentials.password,
        scope: 'openid profile email offline_access'
      })
      
      // Get user from database
      const user = await prisma.user.findUnique({
        where: { email: credentials.email },
        include: { authIdentities: true }
      })
      
      if (!user || !user.isActive) {
        throw new UnauthorizedError('Invalid credentials')
      }
      
      // Check if email is verified
      if (!user.emailVerified) {
        throw new UnauthorizedError('Email not verified')
      }
      
      // Check if MFA is required
      if (user.isMfaEnabled) {
        return this.initiateMfaChallenge(user.id, auth0Response.access_token)
      }
      
      // Create session
      const session = await this.createSession({
        userId: user.id,
        authMethod: 'PASSWORD',
        accessToken: auth0Response.access_token,
        refreshToken: auth0Response.refresh_token,
        rememberMe: credentials.rememberMe
      })
      
      // Log successful login
      await this.logLoginAttempt({
        userId: user.id,
        email: credentials.email,
        authMethod: 'PASSWORD',
        success: true
      })
      
      return {
        success: true,
        user: this.sanitizeUser(user),
        session: {
          token: session.sessionToken,
          expiresAt: session.expiresAt
        }
      }
    } catch (error) {
      // Log failed login
      await this.logLoginAttempt({
        email: credentials.email,
        authMethod: 'PASSWORD',
        success: false,
        errorReason: error.message
      })
      
      throw error
    }
  }

  private async checkRateLimit(email: string) {
    const recentAttempts = await prisma.loginHistory.count({
      where: {
        email,
        success: false,
        attemptedAt: {
          gte: new Date(Date.now() - 15 * 60 * 1000) // Last 15 minutes
        }
      }
    })
    
    if (recentAttempts >= 5) {
      throw new TooManyRequestsError('Too many failed login attempts')
    }
  }
}
```

### 3.3 SSO Login Flow
```typescript
class SSOLoginFlow {
  // Step 1: Initiate SSO
  async initiateSSOLogin(provider: 'google' | 'microsoft' | 'saml', returnUrl?: string) {
    const state = await this.generateSecureState({
      provider,
      returnUrl,
      timestamp: Date.now()
    })
    
    const authorizationUrl = this.getAuthorizationUrl(provider, state)
    
    return { authorizationUrl }
  }

  // Step 2: Handle callback
  async handleSSOCallback(provider: string, code: string, state: string) {
    // Verify state
    const stateData = await this.verifyState(state)
    
    // Exchange code for tokens
    const tokens = await this.exchangeCodeForTokens(provider, code)
    
    // Get user info from provider
    const providerUserInfo = await this.getProviderUserInfo(provider, tokens.access_token)
    
    // Find or create user
    const user = await this.findOrCreateSSOUser(provider, providerUserInfo)
    
    // Create session
    const session = await this.createSession({
      userId: user.id,
      authMethod: this.getAuthMethod(provider),
      accessToken: tokens.access_token,
      refreshToken: tokens.refresh_token
    })
    
    return {
      user,
      session,
      returnUrl: stateData.returnUrl
    }
  }

  private async findOrCreateSSOUser(provider: string, providerInfo: any) {
    // Check if identity already exists
    const existingIdentity = await prisma.authIdentity.findUnique({
      where: {
        provider_providerUserId: {
          provider: provider.toUpperCase(),
          providerUserId: providerInfo.id
        }
      },
      include: { user: true }
    })
    
    if (existingIdentity) {
      // Update last used
      await prisma.authIdentity.update({
        where: { id: existingIdentity.id },
        data: { lastUsedAt: new Date() }
      })
      
      return existingIdentity.user
    }
    
    // Check if user exists with same email
    const existingUser = await prisma.user.findUnique({
      where: { email: providerInfo.email }
    })
    
    if (existingUser) {
      // Link SSO identity to existing user
      await this.linkSSOIdentity(existingUser.id, provider, providerInfo)
      return existingUser
    }
    
    // Create new user with SSO identity
    return this.createSSOUser(provider, providerInfo)
  }

  private async createSSOUser(provider: string, providerInfo: any) {
    return prisma.user.create({
      data: {
        id: generateUserId(),
        email: providerInfo.email,
        emailVerified: true, // SSO emails are pre-verified
        name: providerInfo.name,
        authIdentities: {
          create: {
            provider: provider.toUpperCase(),
            providerUserId: providerInfo.id,
            email: providerInfo.email,
            displayName: providerInfo.name,
            profileData: providerInfo,
            isVerified: true,
            isPrimary: true
          }
        }
      }
    })
  }
}
```

### 3.4 Account Linking
```typescript
class AccountLinkingService {
  // Link email/password to existing SSO account
  async linkEmailPassword(userId: string, email: string, password: string) {
    const user = await this.getUser(userId)
    
    // Verify email doesn't belong to another user
    if (user.email !== email) {
      const existingUser = await prisma.user.findUnique({
        where: { email }
      })
      
      if (existingUser && existingUser.id !== userId) {
        throw new ConflictError('Email already associated with another account')
      }
    }
    
    // Create Auth0 user with password
    const auth0User = await auth0.createUser({
      email,
      password,
      email_verified: user.emailVerified
    })
    
    // Link identity
    await prisma.authIdentity.create({
      data: {
        userId,
        provider: 'EMAIL',
        providerUserId: auth0User.user_id,
        email,
        isVerified: user.emailVerified
      }
    })
    
    return { success: true }
  }

  // Link SSO to existing account
  async linkSSOProvider(userId: string, provider: string, providerInfo: any) {
    // Check if identity already exists
    const existingIdentity = await prisma.authIdentity.findUnique({
      where: {
        provider_providerUserId: {
          provider: provider.toUpperCase(),
          providerUserId: providerInfo.id
        }
      }
    })
    
    if (existingIdentity) {
      if (existingIdentity.userId === userId) {
        throw new ConflictError('Provider already linked to this account')
      } else {
        throw new ConflictError('Provider already linked to another account')
      }
    }
    
    // Link identity
    await prisma.authIdentity.create({
      data: {
        userId,
        provider: provider.toUpperCase(),
        providerUserId: providerInfo.id,
        email: providerInfo.email,
        displayName: providerInfo.name,
        profileData: providerInfo,
        isVerified: true
      }
    })
    
    return { success: true }
  }

  // Unlink auth method
  async unlinkAuthMethod(userId: string, identityId: string) {
    const identities = await prisma.authIdentity.findMany({
      where: { userId }
    })
    
    // Ensure user has at least one auth method remaining
    if (identities.length <= 1) {
      throw new BadRequestError('Cannot remove last authentication method')
    }
    
    const identity = identities.find(i => i.id === identityId)
    if (!identity) {
      throw new NotFoundError('Identity not found')
    }
    
    // Don't allow removing primary identity if it's the only verified one
    if (identity.isPrimary) {
      const otherVerified = identities.find(i => i.id !== identityId && i.isVerified)
      if (!otherVerified) {
        throw new BadRequestError('Cannot remove primary authentication method')
      }
      
      // Make another identity primary
      await prisma.authIdentity.update({
        where: { id: otherVerified.id },
        data: { isPrimary: true }
      })
    }
    
    // Remove identity
    await prisma.authIdentity.delete({
      where: { id: identityId }
    })
    
    // If removing email/password, also remove from Auth0
    if (identity.provider === 'EMAIL') {
      await auth0.deleteUser(identity.providerUserId)
    }
    
    return { success: true }
  }
}
```

## 4. MFA Implementation

### 4.1 MFA Setup
```typescript
class MFAService {
  // Enable MFA for user
  async enableMFA(userId: string, type: MfaType = 'TOTP') {
    const user = await this.getUser(userId)
    
    if (type === 'TOTP') {
      // Generate secret
      const secret = authenticator.generateSecret()
      
      // Generate QR code
      const otpauth = authenticator.keyuri(
        user.email,
        'SEB Dashboard',
        secret
      )
      
      const qrCode = await QRCode.toDataURL(otpauth)
      
      // Store temporarily until verified
      await redis.setex(
        `mfa:setup:${userId}`,
        600, // 10 minutes
        JSON.stringify({ secret, type })
      )
      
      return {
        qrCode,
        secret,
        backupCodes: this.generateBackupCodes()
      }
    }
  }

  // Verify MFA setup
  async verifyMFASetup(userId: string, code: string) {
    const setupData = await redis.get(`mfa:setup:${userId}`)
    if (!setupData) {
      throw new BadRequestError('MFA setup expired')
    }
    
    const { secret, type } = JSON.parse(setupData)
    
    // Verify code
    const isValid = authenticator.verify({
      token: code,
      secret
    })
    
    if (!isValid) {
      throw new BadRequestError('Invalid code')
    }
    
    // Save MFA device
    const backupCodes = this.generateBackupCodes()
    await prisma.mfaDevice.create({
      data: {
        userId,
        deviceName: 'Default',
        type,
        secret: encrypt(secret),
        backupCodes: backupCodes.map(code => hash(code)),
        verifiedAt: new Date(),
        isPrimary: true
      }
    })
    
    // Enable MFA for user
    await prisma.user.update({
      where: { id: userId },
      data: { isMfaEnabled: true }
    })
    
    // Clean up
    await redis.del(`mfa:setup:${userId}`)
    
    return {
      success: true,
      backupCodes
    }
  }

  // Verify MFA code during login
  async verifyMFACode(challengeId: string, code: string) {
    const challenge = await prisma.mfaChallenge.findUnique({
      where: { id: challengeId },
      include: { user: { include: { mfaDevices: true } } }
    })
    
    if (!challenge || challenge.isVerified || challenge.expiresAt < new Date()) {
      throw new BadRequestError('Invalid or expired challenge')
    }
    
    // Check attempts
    if (challenge.attempts >= challenge.maxAttempts) {
      throw new TooManyRequestsError('Too many failed attempts')
    }
    
    // Update attempts
    await prisma.mfaChallenge.update({
      where: { id: challengeId },
      data: { attempts: { increment: 1 } }
    })
    
    // Verify code based on type
    let isValid = false
    
    if (challenge.challengeType === 'TOTP') {
      const device = challenge.user.mfaDevices.find(d => d.isPrimary && d.type === 'TOTP')
      if (device) {
        isValid = authenticator.verify({
          token: code,
          secret: decrypt(device.secret)
        })
      }
    } else if (challenge.challengeType === 'BACKUP_CODE') {
      // Check backup codes
      for (const device of challenge.user.mfaDevices) {
        const hashedCode = hash(code)
        if (device.backupCodes.includes(hashedCode)) {
          // Remove used backup code
          await prisma.mfaDevice.update({
            where: { id: device.id },
            data: {
              backupCodes: device.backupCodes.filter(c => c !== hashedCode)
            }
          })
          isValid = true
          break
        }
      }
    }
    
    if (!isValid) {
      throw new BadRequestError('Invalid code')
    }
    
    // Mark challenge as verified
    await prisma.mfaChallenge.update({
      where: { id: challengeId },
      data: {
        isVerified: true,
        verifiedAt: new Date()
      }
    })
    
    return { success: true }
  }

  private generateBackupCodes(count: number = 8): string[] {
    const codes: string[] = []
    for (let i = 0; i < count; i++) {
      codes.push(
        crypto.randomBytes(4).toString('hex').toUpperCase()
      )
    }
    return codes
  }
}
```

## 5. Password Management

### 5.1 Password Reset Flow
```typescript
class PasswordResetService {
  // Request password reset
  async requestPasswordReset(email: string) {
    const user = await prisma.user.findUnique({
      where: { email },
      include: { authIdentities: true }
    })
    
    if (!user) {
      // Don't reveal if user exists
      return { success: true, message: 'If email exists, reset link sent' }
    }
    
    // Check if user has email/password auth
    const hasEmailAuth = user.authIdentities.some(i => i.provider === 'EMAIL')
    if (!hasEmailAuth) {
      // User only has SSO, can't reset password
      await this.sendSSOOnlyEmail(email)
      return { success: true, message: 'If email exists, reset link sent' }
    }
    
    // Generate reset token
    const token = crypto.randomBytes(32).toString('hex')
    const hashedToken = hash(token)
    
    // Store reset request
    await prisma.passwordReset.create({
      data: {
        userId: user.id,
        token,
        hashedToken,
        expiresAt: new Date(Date.now() + 3600000), // 1 hour
        requestedIp: this.getClientIp()
      }
    })
    
    // Send reset email
    await this.sendPasswordResetEmail(email, token)
    
    return { success: true, message: 'If email exists, reset link sent' }
  }

  // Verify reset token
  async verifyResetToken(token: string) {
    const hashedToken = hash(token)
    
    const resetRequest = await prisma.passwordReset.findFirst({
      where: {
        hashedToken,
        isUsed: false,
        expiresAt: { gt: new Date() }
      }
    })
    
    if (!resetRequest) {
      throw new BadRequestError('Invalid or expired reset token')
    }
    
    return { valid: true, userId: resetRequest.userId }
  }

  // Reset password
  async resetPassword(token: string, newPassword: string) {
    // Verify token
    const { userId } = await this.verifyResetToken(token)
    
    // Validate password strength
    const passwordErrors = validatePasswordStrength(newPassword)
    if (passwordErrors.length > 0) {
      throw new ValidationError('Password does not meet requirements', passwordErrors)
    }
    
    // Update password in Auth0
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: { authIdentities: true }
    })
    
    const emailIdentity = user.authIdentities.find(i => i.provider === 'EMAIL')
    if (emailIdentity) {
      await auth0.updateUser(emailIdentity.providerUserId, {
        password: newPassword
      })
    }
    
    // Mark token as used
    await prisma.passwordReset.updateMany({
      where: {
        hashedToken: hash(token),
        userId
      },
      data: {
        isUsed: true,
        usedAt: new Date()
      }
    })
    
    // Update user
    await prisma.user.update({
      where: { id: userId },
      data: { passwordChangedAt: new Date() }
    })
    
    // Invalidate all sessions
    await this.invalidateAllSessions(userId)
    
    // Send confirmation email
    await this.sendPasswordChangedEmail(user.email)
    
    return { success: true }
  }

  // Change password (authenticated)
  async changePassword(userId: string, currentPassword: string, newPassword: string) {
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: { authIdentities: true }
    })
    
    const emailIdentity = user.authIdentities.find(i => i.provider === 'EMAIL')
    if (!emailIdentity) {
      throw new BadRequestError('No password set for this account')
    }
    
    // Verify current password with Auth0
    try {
      await auth0.authenticate({
        grant_type: 'password',
        username: user.email,
        password: currentPassword
      })
    } catch (error) {
      throw new UnauthorizedError('Current password is incorrect')
    }
    
    // Validate new password
    const passwordErrors = validatePasswordStrength(newPassword)
    if (passwordErrors.length > 0) {
      throw new ValidationError('Password does not meet requirements', passwordErrors)
    }
    
    // Update password
    await auth0.updateUser(emailIdentity.providerUserId, {
      password: newPassword
    })
    
    // Update user record
    await prisma.user.update({
      where: { id: userId },
      data: { passwordChangedAt: new Date() }
    })
    
    return { success: true }
  }
}
```

### 5.2 Password Validation
```typescript
interface PasswordPolicy {
  minLength: number
  requireUppercase: boolean
  requireLowercase: boolean
  requireNumbers: boolean
  requireSpecialChars: boolean
  preventCommon: boolean
  preventUserInfo: boolean
}

const defaultPasswordPolicy: PasswordPolicy = {
  minLength: 8,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: true,
  preventCommon: true,
  preventUserInfo: true
}

function validatePasswordStrength(
  password: string,
  userInfo?: { email?: string; name?: string },
  policy: PasswordPolicy = defaultPasswordPolicy
): string[] {
  const errors: string[] = []
  
  // Length check
  if (password.length < policy.minLength) {
    errors.push(`Password must be at least ${policy.minLength} characters`)
  }
  
  // Character requirements
  if (policy.requireUppercase && !/[A-Z]/.test(password)) {
    errors.push('Password must contain at least one uppercase letter')
  }
  
  if (policy.requireLowercase && !/[a-z]/.test(password)) {
    errors.push('Password must contain at least one lowercase letter')
  }
  
  if (policy.requireNumbers && !/[0-9]/.test(password)) {
    errors.push('Password must contain at least one number')
  }
  
  if (policy.requireSpecialChars && !/[!@#$%^&*(),.?":{}|<>]/.test(password)) {
    errors.push('Password must contain at least one special character')
  }
  
  // Common password check
  if (policy.preventCommon && commonPasswords.includes(password.toLowerCase())) {
    errors.push('Password is too common')
  }
  
  // User info check
  if (policy.preventUserInfo && userInfo) {
    const lowerPassword = password.toLowerCase()
    
    if (userInfo.email && lowerPassword.includes(userInfo.email.split('@')[0].toLowerCase())) {
      errors.push('Password cannot contain your email')
    }
    
    if (userInfo.name && lowerPassword.includes(userInfo.name.toLowerCase())) {
      errors.push('Password cannot contain your name')
    }
  }
  
  return errors
}
```

## 6. Session Management

### 6.1 Session Service
```typescript
class SessionService {
  // Create session
  async createSession(data: {
    userId: string
    authMethod: AuthMethod
    accessToken: string
    refreshToken?: string
    rememberMe?: boolean
    ipAddress?: string
    userAgent?: string
  }) {
    const expiresAt = data.rememberMe
      ? new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) // 30 days
      : new Date(Date.now() + 24 * 60 * 60 * 1000) // 24 hours
    
    const sessionToken = this.generateSessionToken()
    
    const session = await prisma.userSession.create({
      data: {
        userId: data.userId,
        sessionToken,
        refreshToken: data.refreshToken,
        authMethod: data.authMethod,
        ipAddress: data.ipAddress,
        userAgent: data.userAgent,
        deviceInfo: await this.getDeviceInfo(data.userAgent),
        expiresAt
      }
    })
    
    // Update user last login
    await prisma.user.update({
      where: { id: data.userId },
      data: {
        lastLoginAt: new Date(),
        lastLoginIp: data.ipAddress,
        lastLoginMethod: data.authMethod
      }
    })
    
    // Store in cache for fast lookup
    await redis.setex(
      `session:${sessionToken}`,
      Math.floor((expiresAt.getTime() - Date.now()) / 1000),
      JSON.stringify({
        userId: data.userId,
        sessionId: session.id
      })
    )
    
    return session
  }

  // Validate session
  async validateSession(sessionToken: string): Promise<SessionValidation> {
    // Check cache first
    const cached = await redis.get(`session:${sessionToken}`)
    if (cached) {
      const { userId, sessionId } = JSON.parse(cached)
      return { valid: true, userId, sessionId }
    }
    
    // Check database
    const session = await prisma.userSession.findUnique({
      where: { sessionToken },
      include: { user: true }
    })
    
    if (!session || !session.isActive || session.expiresAt < new Date()) {
      return { valid: false }
    }
    
    if (!session.user.isActive) {
      return { valid: false, reason: 'User account deactivated' }
    }
    
    // Update last activity
    await prisma.userSession.update({
      where: { id: session.id },
      data: { lastActivityAt: new Date() }
    })
    
    // Cache for future requests
    await redis.setex(
      `session:${sessionToken}`,
      Math.floor((session.expiresAt.getTime() - Date.now()) / 1000),
      JSON.stringify({
        userId: session.userId,
        sessionId: session.id
      })
    )
    
    return {
      valid: true,
      userId: session.userId,
      sessionId: session.id,
      user: session.user
    }
  }

  // Refresh session
  async refreshSession(refreshToken: string) {
    const session = await prisma.userSession.findUnique({
      where: { refreshToken },
      include: { user: true }
    })
    
    if (!session || !session.isActive) {
      throw new UnauthorizedError('Invalid refresh token')
    }
    
    // Refresh Auth0 tokens
    const auth0Tokens = await auth0.refreshToken(refreshToken)
    
    // Create new session
    const newSession = await this.createSession({
      userId: session.userId,
      authMethod: session.authMethod,
      accessToken: auth0Tokens.access_token,
      refreshToken: auth0Tokens.refresh_token,
      ipAddress: session.ipAddress,
      userAgent: session.userAgent
    })
    
    // Invalidate old session
    await this.invalidateSession(session.id)
    
    return newSession
  }

  // Invalidate session
  async invalidateSession(sessionId: string) {
    const session = await prisma.userSession.update({
      where: { id: sessionId },
      data: { isActive: false }
    })
    
    // Remove from cache
    await redis.del(`session:${session.sessionToken}`)
  }

  // Get active sessions
  async getActiveSessions(userId: string) {
    return prisma.userSession.findMany({
      where: {
        userId,
        isActive: true,
        expiresAt: { gt: new Date() }
      },
      orderBy: { lastActivityAt: 'desc' }
    })
  }

  // Invalidate all sessions
  async invalidateAllSessions(userId: string, exceptSessionId?: string) {
    const sessions = await prisma.userSession.findMany({
      where: {
        userId,
        isActive: true,
        id: exceptSessionId ? { not: exceptSessionId } : undefined
      }
    })
    
    // Invalidate in database
    await prisma.userSession.updateMany({
      where: {
        userId,
        id: exceptSessionId ? { not: exceptSessionId } : undefined
      },
      data: { isActive: false }
    })
    
    // Remove from cache
    for (const session of sessions) {
      await redis.del(`session:${session.sessionToken}`)
    }
  }

  private generateSessionToken(): string {
    return crypto.randomBytes(32).toString('hex')
  }

  private async getDeviceInfo(userAgent?: string) {
    if (!userAgent) return null
    
    const parser = new UAParser(userAgent)
    const result = parser.getResult()
    
    return {
      browser: result.browser.name,
      browserVersion: result.browser.version,
      os: result.os.name,
      osVersion: result.os.version,
      device: result.device.type || 'desktop'
    }
  }
}
```

## 7. API Endpoints

### 7.1 Authentication Endpoints
```typescript
// Registration
POST /api/v1/auth/register
{
  "email": "user@example.com",
  "password": "SecurePassword123!",
  "name": "John Doe",
  "nameJa": "ジョン・ドー",
  "locale": "ja"
}

// Email/Password Login
POST /api/v1/auth/login
{
  "email": "user@example.com",
  "password": "SecurePassword123!",
  "rememberMe": true
}

// SSO Login Initiation
GET /api/v1/auth/sso/{provider}?returnUrl=/dashboard

// SSO Callback
GET /api/v1/auth/sso/{provider}/callback?code=...&state=...

// Logout
POST /api/v1/auth/logout
{
  "sessionToken": "...",
  "logoutAllDevices": false
}

// Refresh Token
POST /api/v1/auth/refresh
{
  "refreshToken": "..."
}
```

### 7.2 Account Management Endpoints
```typescript
// Get current user with auth methods
GET /api/v1/auth/me
Response:
{
  "user": {
    "id": "...",
    "email": "user@example.com",
    "name": "John Doe",
    "authMethods": [
      {
        "id": "...",
        "provider": "EMAIL",
        "email": "user@example.com",
        "isPrimary": true,
        "isVerified": true
      },
      {
        "id": "...",
        "provider": "GOOGLE",
        "email": "user@gmail.com",
        "displayName": "John Doe",
        "isPrimary": false,
        "isVerified": true
      }
    ],
    "mfaEnabled": false
  }
}

// Link auth method
POST /api/v1/auth/link/{provider}

// Unlink auth method
DELETE /api/v1/auth/link/{identityId}

// Change password
POST /api/v1/auth/change-password
{
  "currentPassword": "OldPassword123!",
  "newPassword": "NewPassword456!"
}

// Request password reset
POST /api/v1/auth/forgot-password
{
  "email": "user@example.com"
}

// Reset password
POST /api/v1/auth/reset-password
{
  "token": "...",
  "newPassword": "NewPassword456!"
}
```

### 7.3 MFA Endpoints
```typescript
// Enable MFA
POST /api/v1/auth/mfa/enable
{
  "type": "TOTP"
}
Response:
{
  "qrCode": "data:image/png;base64,...",
  "secret": "JBSWY3DPEHPK3PXP",
  "backupCodes": ["ABCD1234", "EFGH5678", ...]
}

// Verify MFA setup
POST /api/v1/auth/mfa/verify
{
  "code": "123456"
}

// Disable MFA
POST /api/v1/auth/mfa/disable
{
  "code": "123456"
}

// Verify MFA during login
POST /api/v1/auth/mfa/challenge/{challengeId}
{
  "code": "123456"
}
```

### 7.4 Session Management Endpoints
```typescript
// Get active sessions
GET /api/v1/auth/sessions

// Revoke session
DELETE /api/v1/auth/sessions/{sessionId}

// Revoke all other sessions
POST /api/v1/auth/sessions/revoke-others
```

## 8. Security Measures

### 8.1 Security Headers
```typescript
const securityHeaders = {
  'Strict-Transport-Security': 'max-age=63072000; includeSubDomains; preload',
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Content-Security-Policy': "default-src 'self'; script-src 'self' 'unsafe-inline' https://apis.google.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self' https://api.auth0.com"
}
```

### 8.2 Rate Limiting
```typescript
const rateLimits = {
  login: {
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // 5 attempts
    skipSuccessfulRequests: true
  },
  passwordReset: {
    windowMs: 60 * 60 * 1000, // 1 hour
    max: 3 // 3 requests
  },
  registration: {
    windowMs: 60 * 60 * 1000, // 1 hour
    max: 5 // 5 registrations
  },
  mfaVerification: {
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 10 // 10 attempts
  }
}
```

### 8.3 Security Monitoring
```typescript
class SecurityMonitor {
  // Monitor suspicious activities
  async detectSuspiciousActivity(userId: string) {
    const recentLogins = await prisma.loginHistory.findMany({
      where: {
        userId,
        attemptedAt: {
          gte: new Date(Date.now() - 24 * 60 * 60 * 1000) // Last 24 hours
        }
      }
    })
    
    // Check for multiple locations
    const locations = new Set(recentLogins.map(l => l.location).filter(Boolean))
    if (locations.size > 3) {
      await this.alertSuspiciousActivity(userId, 'Multiple login locations detected')
    }
    
    // Check for failed attempts
    const failedAttempts = recentLogins.filter(l => !l.success).length
    if (failedAttempts > 10) {
      await this.alertSuspiciousActivity(userId, 'Multiple failed login attempts')
    }
  }

  // Alert user and admins
  async alertSuspiciousActivity(userId: string, reason: string) {
    // Send email to user
    await emailService.send({
      to: user.email,
      subject: 'Suspicious activity detected',
      template: 'suspicious-activity',
      data: { reason, timestamp: new Date() }
    })
    
    // Log for admin review
    await prisma.securityAlert.create({
      data: {
        userId,
        alertType: 'SUSPICIOUS_ACTIVITY',
        reason,
        timestamp: new Date()
      }
    })
  }
}
```

## 9. Frontend Integration

### 9.1 Auth Context
```typescript
// contexts/AuthContext.tsx
interface AuthContextValue {
  user: User | null
  isAuthenticated: boolean
  isLoading: boolean
  authMethods: AuthIdentity[]
  login: (credentials: LoginCredentials) => Promise<void>
  loginWithSSO: (provider: string) => Promise<void>
  logout: (options?: LogoutOptions) => Promise<void>
  register: (data: RegistrationData) => Promise<void>
  linkAuthMethod: (provider: string) => Promise<void>
  unlinkAuthMethod: (identityId: string) => Promise<void>
}

export const AuthProvider: React.FC = ({ children }) => {
  const [user, setUser] = useState<User | null>(null)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    // Check for existing session
    checkSession()
  }, [])

  const login = async (credentials: LoginCredentials) => {
    try {
      const response = await api.post('/auth/login', credentials)
      
      if (response.data.requiresMFA) {
        // Redirect to MFA page
        router.push(`/auth/mfa?challenge=${response.data.challengeId}`)
        return
      }
      
      setUser(response.data.user)
      storeSession(response.data.session)
      router.push('/dashboard')
    } catch (error) {
      handleAuthError(error)
    }
  }

  const loginWithSSO = async (provider: string) => {
    const response = await api.get(`/auth/sso/${provider}`)
    window.location.href = response.data.authorizationUrl
  }

  // ... rest of implementation
}
```

### 9.2 Auth Components
```typescript
// components/auth/LoginForm.tsx
export const LoginForm: React.FC = () => {
  const { login, loginWithSSO } = useAuth()
  const [isLoading, setIsLoading] = useState(false)

  const onSubmit = async (data: LoginFormData) => {
    setIsLoading(true)
    try {
      await login(data)
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <TextField
        name="email"
        type="email"
        label={t('auth.email')}
        required
      />
      
      <TextField
        name="password"
        type="password"
        label={t('auth.password')}
        required
      />
      
      <Checkbox
        name="rememberMe"
        label={t('auth.rememberMe')}
      />
      
      <Button type="submit" loading={isLoading}>
        {t('auth.login')}
      </Button>
      
      <Divider>{t('auth.or')}</Divider>
      
      <SSOButtons>
        <Button
          variant="outline"
          onClick={() => loginWithSSO('google')}
          icon={<GoogleIcon />}
        >
          {t('auth.continueWithGoogle')}
        </Button>
        
        <Button
          variant="outline"
          onClick={() => loginWithSSO('microsoft')}
          icon={<MicrosoftIcon />}
        >
          {t('auth.continueWithMicrosoft')}
        </Button>
      </SSOButtons>
      
      <Link href="/auth/forgot-password">
        {t('auth.forgotPassword')}
      </Link>
    </form>
  )
}

// components/auth/AccountLinking.tsx
export const AccountLinking: React.FC = () => {
  const { user, authMethods, linkAuthMethod, unlinkAuthMethod } = useAuth()
  
  const availableProviders = ['EMAIL', 'GOOGLE', 'MICROSOFT'].filter(
    provider => !authMethods.some(m => m.provider === provider)
  )

  return (
    <div>
      <h3>{t('account.linkedAccounts')}</h3>
      
      <LinkedAccountsList>
        {authMethods.map(method => (
          <LinkedAccount key={method.id}>
            <ProviderIcon provider={method.provider} />
            <div>
              <div>{method.displayName || method.email}</div>
              <div>{method.provider}</div>
            </div>
            {authMethods.length > 1 && (
              <Button
                variant="ghost"
                size="sm"
                onClick={() => unlinkAuthMethod(method.id)}
              >
                {t('account.unlink')}
              </Button>
            )}
          </LinkedAccount>
        ))}
      </LinkedAccountsList>
      
      {availableProviders.length > 0 && (
        <div>
          <h4>{t('account.linkNewAccount')}</h4>
          {availableProviders.map(provider => (
            <Button
              key={provider}
              variant="outline"
              onClick={() => linkAuthMethod(provider)}
            >
              {t(`account.linkWith${provider}`)}
            </Button>
          ))}
        </div>
      )}
    </div>
  )
}
```

## 10. Migration Guide

### 10.1 Existing Users Migration
```typescript
// Migration script for existing users
async function migrateExistingUsers() {
  const users = await prisma.user.findMany()
  
  for (const user of users) {
    // Create auth identity for existing email users
    if (user.email) {
      await prisma.authIdentity.create({
        data: {
          userId: user.id,
          provider: 'EMAIL',
          providerUserId: user.id, // Use existing ID
          email: user.email,
          isPrimary: true,
          isVerified: user.emailVerified
        }
      })
    }
  }
}
```

### 10.2 Auth0 Configuration
```javascript
// Auth0 tenant configuration
{
  "tenant": "seb-dashboard",
  "environment": "production",
  "connections": [
    {
      "name": "Username-Password-Authentication",
      "strategy": "auth0",
      "enabled_clients": ["seb-dashboard-web"],
      "options": {
        "password_policy": "good",
        "password_history": {
          "enable": true,
          "size": 5
        },
        "password_dictionary": {
          "enable": true
        },
        "brute_force_protection": true
      }
    },
    {
      "name": "google-oauth2",
      "strategy": "google-oauth2",
      "enabled_clients": ["seb-dashboard-web"],
      "options": {
        "client_id": "${GOOGLE_CLIENT_ID}",
        "client_secret": "${GOOGLE_CLIENT_SECRET}",
        "scope": ["email", "profile"]
      }
    }
  ],
  "rules": [
    {
      "name": "Add user metadata",
      "script": "function (user, context, callback) {\n  user.app_metadata = user.app_metadata || {};\n  user.user_metadata = user.user_metadata || {};\n  \n  context.idToken['https://seb-dashboard.com/metadata'] = user.user_metadata;\n  context.idToken['https://seb-dashboard.com/roles'] = user.app_metadata.roles || [];\n  \n  callback(null, user, context);\n}"
    }
  ]
}
```