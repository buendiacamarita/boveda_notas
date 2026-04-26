# Article Synchronization Implementation

## Overview

This document describes the implementation of article synchronization after login in the MIDAS.Mobile application. The feature ensures that all articles from the database are synchronized and stored in the local SQLite database after a user logs in.

## Implementation Details

### 1. Created a Synchronization Service

A new service file was created at `src/domain/services/shared/syncService.ts` to handle the synchronization of articles. This service:

- Fetches all articles from the API using the existing `productsService.getAll()` function
- Stores the articles in the local SQLite database using the existing `articulosRepository.syncArticulos()` function
- Includes error handling with try/catch
- Returns the result of the synchronization

```typescript
import { getAll as getAllProducts } from '../../../infraestructure/api/services/Products/productsService';
import { articulosRepository } from '../../../infraestructure/repositories/articulos.repository';

/**
 * Synchronizes all articles from the API to the local SQLite database.
 * This function fetches all articles from the API and stores them in the local database.
 * @returns A promise that resolves when the synchronization is complete.
 */
export const syncArticles = async (): Promise<string> => {
    try {
        // Fetch all articles from the API
        const articles = await getAllProducts();
        
        // Store the articles in the local database
        const repository = articulosRepository();
        const result = await repository.syncArticulos(articles);
        
        return result;
    } catch (error) {
        console.error('Error synchronizing articles:', error);
        throw error;
    }
};
```

### 2. Integrated Synchronization with Login

The login function in `src/infraestructure/contexts/AuthContext.tsx` was modified to call the `syncArticles` function after successful login:

```typescript
const login = async (email:string, password:string): Promise<void> => {
    try {
        const response = await axios.post(`${API_LOGIN}auth/authenticate/`, {Email: email, Password: password, refreshToken: "", grantType: 'password' });
        const { token, refreshToken, userName } = response.data.data;
        await setTokens(token, refreshToken)
        setUser(userName);
        
        // Synchronize all articles from the API to the local database
        try {
            console.log('Starting article synchronization...');
            await syncArticles();
            console.log('Article synchronization completed successfully');
        } catch (syncError) {
            console.error('Error during article synchronization:', syncError);
            // We don't throw the error here to avoid blocking the login process
            // The user can still use the app even if synchronization fails
        }
    } catch (error) {
        throw error;
    }
};
```

The synchronization happens in a nested try/catch block to handle synchronization errors separately from login errors. This ensures that the user can still use the app even if synchronization fails.

## Testing

To test the implementation:

1. Build and run the application
2. Log in with valid credentials
3. Check the console logs for synchronization messages:
   - "Starting article synchronization..."
   - "Article synchronization completed successfully"
4. Verify that articles are stored in the local database by:
   - Navigating to a screen that displays articles
   - Using the app in offline mode to confirm articles are available

## Troubleshooting

If synchronization fails, check the console logs for error messages. Common issues might include:

- Network connectivity problems
- API endpoint issues
- Database access or permission issues

The error will be logged with the message "Error during article synchronization:" followed by the specific error details.
