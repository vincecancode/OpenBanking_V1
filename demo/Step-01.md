# Step 1: Basic OAuth 2.0 Server

## Overview
This step creates a simple OAuth 2.0 authorization server that demonstrates the fundamental concepts of OAuth 2.0 Authorization Code flow. We'll build a working server that can issue access tokens and handle basic authentication flows.

## What You'll Learn
- OAuth 2.0 Authorization Code flow
- Basic token issuance and validation
- Client registration and management
- Simple consent flow
- HTTP server setup with Node.js

## How this program works

This is a barebones OAuth 2.0 Authorization Server demo in Node.js. It implements the Authorization Code flow, which is the standard for web and mobile apps. The server:

- Lets you register OAuth clients (apps that want to access APIs on behalf of users)
- Stores users, clients, authorization codes, and access tokens in memory (not for production)
- Handles `/authorize` to issue authorization codes after user login/consent
- Handles `/token` to exchange codes for access tokens
- Validates client credentials and redirect URIs
- Provides a simple HTML form for demo client registration
- Exposes a status endpoint for monitoring

It's intentionally simple: no refresh tokens, no PKCE, no OpenID Connect, no persistent storage, no HTTPS. It's for learning OAuth basics, not for real-world use. You can run it, register a client, and walk through the full OAuth 2.0 flow with curl or a browser.

## Prerequisites
- Node.js 18+ installed
- Basic understanding of HTTP and JavaScript
- Terminal/command line access

## Implementation

### 1. Project Setup

Create the project structure:
```bash
mkdir step1
cd step1
npm init -y
```

Install dependencies:
```bash
npm install express cors body-parser crypto-js uuid
```

### 2. Basic OAuth Server Implementation

Create `basic-oauth-server.js`:

```javascript
const express = require('express');
const cors = require('cors');
const bodyParser = require('body-parser');
const crypto = require('crypto');
const { v4: uuidv4 } = require('uuid');

const app = express();
const PORT = 3000;

// Middleware
app.use(cors());
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

// In-memory storage (for demo purposes)
const clients = new Map();
const authorizationCodes = new Map();
const accessTokens = new Map();
const users = new Map();

// Sample users
users.set('user1', { id: 'user1', username: 'alice', password: 'password123' });
users.set('user2', { id: 'user2', username: 'bob', password: 'password456' });

// Sample client (registered beforehand)
clients.set('demo-client', {
    clientId: 'demo-client',
    clientSecret: 'demo-secret',
    redirectUris: ['http://localhost:3001/callback'],
    grantTypes: ['authorization_code'],
    responseTypes: ['code'],
    scope: 'read:accounts write:payments'
});

// Helper function to generate tokens
function generateToken() {
    return crypto.randomBytes(32).toString('hex');
}

// Helper function to validate client
function validateClient(clientId, clientSecret) {
    const client = clients.get(clientId);
    return client && client.clientSecret === clientSecret;
}

// Helper function to validate redirect URI
function validateRedirectUri(clientId, redirectUri) {
    const client = clients.get(clientId);
    return client && client.redirectUris.includes(redirectUri);
}

// 1. Authorization Endpoint
app.get('/oauth2/authorize', (req, res) => {
    const { client_id, redirect_uri, response_type, scope, state } = req.query;
    
    console.log('Authorization request:', req.query);
    
    // Validate required parameters
    if (!client_id || !redirect_uri || !response_type) {
        return res.status(400).json({
            error: 'invalid_request',
            error_description: 'Missing required parameters'
        });
    }
    
    // Validate client
    if (!clients.has(client_id)) {
        return res.status(400).json({
            error: 'invalid_client',
            error_description: 'Unknown client'
        });
    }
    
    // Validate redirect URI
    if (!validateRedirectUri(client_id, redirect_uri)) {
        return res.status(400).json({
            error: 'invalid_request',
            error_description: 'Invalid redirect URI'
        });
    }
    
    // Check if user is already authenticated (simplified)
    const isAuthenticated = req.query.authenticated === 'true';
    
    if (!isAuthenticated) {
        // Show login form
        return res.send(`
            <html>
                <head><title>Login - OAuth Demo</title></head>
                <body>
                    <h2>OAuth 2.0 Demo - Login</h2>
                    <form method="post" action="/oauth2/login">
                        <input type="hidden" name="client_id" value="${client_id}">
                        <input type="hidden" name="redirect_uri" value="${redirect_uri}">
                        <input type="hidden" name="response_type" value="${response_type}">
                        <input type="hidden" name="scope" value="${scope || ''}">
                        <input type="hidden" name="state" value="${state || ''}">
                        
                        <div>
                            <label>Username:</label>
                            <input type="text" name="username" required>
                        </div>
                        <div>
                            <label>Password:</label>
                            <input type="password" name="password" required>
                        </div>
                        <div>
                            <button type="submit">Login</button>
                        </div>
                    </form>
                </body>
            </html>
        `);
    }
    
    // Show consent form
    const client = clients.get(client_id);
    return res.send(`
        <html>
            <head><title>Consent - OAuth Demo</title></head>
            <body>
                <h2>OAuth 2.0 Demo - Consent</h2>
                <p>Client: <strong>${client.clientId}</strong></p>
                <p>Requested Scopes: <strong>${scope || 'No scopes requested'}</strong></p>
                
                <form method="post" action="/oauth2/consent">
                    <input type="hidden" name="client_id" value="${client_id}">
                    <input type="hidden" name="redirect_uri" value="${redirect_uri}">
                    <input type="hidden" name="response_type" value="${response_type}">
                    <input type="hidden" name="scope" value="${scope || ''}">
                    <input type="hidden" name="state" value="${state || ''}">
                    <input type="hidden" name="user_id" value="user1">
                    
                    <div>
                        <label>
                            <input type="checkbox" name="consent" value="true" checked>
                            I consent to the requested permissions
                        </label>
                    </div>
                    <div>
                        <button type="submit" name="action" value="approve">Approve</button>
                        <button type="submit" name="action" value="deny">Deny</button>
                    </div>
                </form>
            </body>
        </html>
    `);
});

// 2. Login Endpoint
app.post('/oauth2/login', (req, res) => {
    const { username, password, client_id, redirect_uri, response_type, scope, state } = req.body;
    
    // Simple user validation
    const user = Array.from(users.values()).find(u => u.username === username);
    
    if (!user || user.password !== password) {
        return res.status(401).send(`
            <html>
                <head><title>Login Failed</title></head>
                <body>
                    <h2>Login Failed</h2>
                    <p>Invalid username or password</p>
                    <a href="/oauth2/authorize?client_id=${client_id}&redirect_uri=${redirect_uri}&response_type=${response_type}&scope=${scope}&state=${state}&authenticated=true">Try Again</a>
                </body>
            </html>
        `);
    }
    
    // Redirect to consent with authenticated flag
    const params = new URLSearchParams({
        client_id,
        redirect_uri,
        response_type,
        scope: scope || '',
        state: state || '',
        authenticated: 'true',
        user_id: user.id
    });
    
    res.redirect(`/oauth2/authorize?${params.toString()}`);
});

// 3. Consent Endpoint
app.post('/oauth2/consent', (req, res) => {
    const { client_id, redirect_uri, response_type, scope, state, user_id, action } = req.body;
    
    if (action === 'deny') {
        return res.redirect(`${redirect_uri}?error=access_denied&state=${state || ''}`);
    }
    
    if (action === 'approve') {
        // Generate authorization code
        const authCode = generateToken();
        authorizationCodes.set(authCode, {
            code: authCode,
            clientId: client_id,
            userId: user_id,
            scope: scope || '',
            redirectUri: redirect_uri,
            expiresAt: Date.now() + 600000 // 10 minutes
        });
        
        console.log('Authorization code generated:', authCode);
        
        // Redirect back to client with authorization code
        const params = new URLSearchParams({
            code: authCode,
            state: state || ''
        });
        
        res.redirect(`${redirect_uri}?${params.toString()}`);
    }
});

// 4. Token Endpoint
app.post('/oauth2/token', (req, res) => {
    const { grant_type, code, redirect_uri, client_id, client_secret } = req.body;
    
    console.log('Token request:', req.body);
    
    // Validate grant type
    if (grant_type !== 'authorization_code') {
        return res.status(400).json({
            error: 'unsupported_grant_type',
            error_description: 'Only authorization_code grant type is supported'
        });
    }
    
    // Validate client credentials
    if (!validateClient(client_id, client_secret)) {
        return res.status(401).json({
            error: 'invalid_client',
            error_description: 'Invalid client credentials'
        });
    }
    
    // Validate authorization code
    const authCodeData = authorizationCodes.get(code);
    if (!authCodeData) {
        return res.status(400).json({
            error: 'invalid_grant',
            error_description: 'Invalid authorization code'
        });
    }
    
    // Check if code is expired
    if (Date.now() > authCodeData.expiresAt) {
        authorizationCodes.delete(code);
        return res.status(400).json({
            error: 'invalid_grant',
            error_description: 'Authorization code expired'
        });
    }
    
    // Validate redirect URI
    if (redirect_uri !== authCodeData.redirectUri) {
        return res.status(400).json({
            error: 'invalid_grant',
            error_description: 'Redirect URI mismatch'
        });
    }
    
    // Generate access token
    const accessToken = generateToken();
    const tokenData = {
        token: accessToken,
        clientId: client_id,
        userId: authCodeData.userId,
        scope: authCodeData.scope,
        expiresAt: Date.now() + 3600000 // 1 hour
    };
    
    accessTokens.set(accessToken, tokenData);
    
    // Clean up authorization code (one-time use)
    authorizationCodes.delete(code);
    
    console.log('Access token generated:', accessToken);
    
    // Return token response
    res.json({
        access_token: accessToken,
        token_type: 'Bearer',
        expires_in: 3600,
        scope: authCodeData.scope
    });
});

// 5. Token Introspection Endpoint
app.post('/oauth2/introspect', (req, res) => {
    const { token } = req.body;
    
    const tokenData = accessTokens.get(token);
    
    if (!tokenData) {
        return res.json({ active: false });
    }
    
    if (Date.now() > tokenData.expiresAt) {
        accessTokens.delete(token);
        return res.json({ active: false });
    }
    
    res.json({
        active: true,
        client_id: tokenData.clientId,
        user_id: tokenData.userId,
        scope: tokenData.scope,
        exp: Math.floor(tokenData.expiresAt / 1000)
    });
});

// 6. Client Registration Endpoint
app.post('/oauth2/register', (req, res) => {
    const { client_name, redirect_uris, grant_types, response_types } = req.body;
    
    const clientId = generateToken().substring(0, 16);
    const clientSecret = generateToken();
    
    const client = {
        clientId,
        clientSecret,
        clientName: client_name,
        redirectUris: redirect_uris || [],
        grantTypes: grant_types || ['authorization_code'],
        responseTypes: response_types || ['code'],
        scope: 'read:accounts write:payments'
    };
    
    clients.set(clientId, client);
    
    console.log('New client registered:', clientId);
    
    res.json({
        client_id: clientId,
        client_secret: clientSecret,
        client_name: client_name,
        redirect_uris: redirect_uris,
        grant_types: grant_types,
        response_types: response_types
    });
});

// 7. Demo Client Registration
app.get('/oauth2/demo/register', (req, res) => {
    res.send(`
        <html>
            <head><title>Client Registration - OAuth Demo</title></head>
            <body>
                <h2>OAuth 2.0 Demo - Client Registration</h2>
                <form method="post" action="/oauth2/register">
                    <div>
                        <label>Client Name:</label>
                        <input type="text" name="client_name" value="Demo App" required>
                    </div>
                    <div>
                        <label>Redirect URIs (one per line):</label>
                        <textarea name="redirect_uris" rows="3">http://localhost:3001/callback</textarea>
                    </div>
                    <div>
                        <label>Grant Types:</label>
                        <input type="text" name="grant_types" value="authorization_code" required>
                    </div>
                    <div>
                        <label>Response Types:</label>
                        <input type="text" name="response_types" value="code" required>
                    </div>
                    <div>
                        <button type="submit">Register Client</button>
                    </div>
                </form>
            </body>
        </html>
    `);
});

// 8. Server Status
app.get('/oauth2/status', (req, res) => {
    res.json({
        server: 'OAuth 2.0 Demo Server',
        version: '1.0.0',
        clients: clients.size,
        activeTokens: accessTokens.size,
        pendingCodes: authorizationCodes.size
    });
});

// Start server
app.listen(PORT, () => {
    console.log(`OAuth 2.0 Server running on http://localhost:${PORT}`);
    console.log(`Demo client registration: http://localhost:${PORT}/oauth2/demo/register`);
    console.log(`Server status: http://localhost:${PORT}/oauth2/status`);
});
```

### 3. Package.json

Create `package.json`:

```json
{
  "name": "step1-basic-oauth-server",
  "version": "1.0.0",
  "description": "Basic OAuth 2.0 Authorization Server - Step 1",
  "main": "basic-oauth-server.js",
  "scripts": {
    "start": "node basic-oauth-server.js",
    "dev": "nodemon basic-oauth-server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "body-parser": "^1.20.2",
    "crypto-js": "^4.1.1",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  },
  "keywords": ["oauth2", "authorization", "demo", "fapi"],
  "author": "FAPI Demo",
  "license": "MIT"
}
```

## Running the Demo

### 1. Start the Server
```bash
cd step1
npm install
npm start
```

### 2. Test the OAuth Flow

#### Step 2a: Register a New Client
1. Open: http://localhost:3000/oauth2/demo/register
2. Fill in the form and click "Register Client"
3. Note the `client_id` and `client_secret` returned

#### Step 2b: Test Authorization Flow
1. Open: http://localhost:3000/oauth2/authorize?client_id=demo-client&redirect_uri=http://localhost:3001/callback&response_type=code&scope=read:accounts
2. Login with: username=`alice`, password=`password123`
3. Approve the consent
4. You'll be redirected with an authorization code

#### Step 2c: Exchange Code for Token
```bash
curl -X POST http://localhost:3000/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&code=YOUR_AUTH_CODE&redirect_uri=http://localhost:3001/callback&client_id=demo-client&client_secret=demo-secret"
```

#### Step 2d: Introspect Token
```bash
curl -X POST http://localhost:3000/oauth2/introspect \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "token=YOUR_ACCESS_TOKEN"
```

## Key Concepts Demonstrated

### 1. **Authorization Code Flow**
- User authentication
- Consent management
- Authorization code generation
- Token exchange

### 2. **Client Management**
- Client registration
- Client validation
- Redirect URI validation

### 3. **Token Management**
- Access token generation
- Token introspection
- Token expiration
- One-time use codes

### 4. **Security Basics**
- Client authentication
- Redirect URI validation
- Token expiration
- Code expiration

## What's Missing (Will be added in later steps)

- **PKCE Support**: For public clients
- **mTLS**: Mutual TLS authentication
- **JWT Tokens**: Signed tokens
- **Request Signing**: JAR (JWT Authorization Requests)
- **Advanced Security**: FAPI 2.0 compliance
- **Monitoring**: Audit logging
- **Database**: Persistent storage

## Next Steps

In Step 2, we'll add:
- Resource server with protected endpoints
- API gateway functionality
- Basic CORS configuration
- Token validation middleware

## Troubleshooting

### Common Issues

1. **Port already in use**: Change PORT in the code
2. **CORS errors**: Ensure CORS middleware is properly configured
3. **Token not found**: Check if token hasn't expired
4. **Invalid client**: Verify client_id and client_secret

### Debug Tips

- Check server logs for detailed information
- Use browser developer tools to inspect requests
- Verify all required parameters are present
- Check token expiration times

## Security Notes

⚠️ **This is a demo implementation with minimal security!**

- No HTTPS (use in development only)
- In-memory storage (data lost on restart)
- Simple password validation
- No rate limiting
- No audit logging

These security features will be added in subsequent steps.
