# Chapter 6: Global State Management (AppStore)

In the [previous chapter](05_frontend_application_core___routing_.md), we learned how the HIP frontend application organizes its different pages and allows navigation using the **Frontend Application Core & Routing**. We saw how components like the `Sidebar` and main content pages like `Files` are displayed based on the URL.

But how do these different parts communicate or share information? For example, the Sidebar might need to display the logged-in user's name, and the [Remote Desktop & App Management](04_remote_desktop___app_management_.md) page needs to know which virtual desktops belong to that user. If each component had to ask the server for this information independently, it would be inefficient and hard to keep consistent.

## The Problem: Sharing Information Across Components

Imagine your application is like a busy research lab. You have different rooms (components) for different tasks:

*   The **Reception Desk (Sidebar)** needs to know who is currently logged in.
*   The **Project Office (Project Page)** needs the list of projects the logged-in user belongs to.
*   The **Equipment Room (Desktop Manager)** needs the list of virtual machines assigned to the user.

How do we make sure everyone in the lab has access to the *same*, *up-to-date* information without running around asking the lab director (the server) every single time? We need a central bulletin board or a shared information system.

This is where **Global State Management** comes in.

## What is Global State Management (AppStore)?

Think of HIP's Global State Management system, often called the **AppStore** (though implemented using React Context in the code), as the central control room or the main bulletin board for the frontend application.

*   **Purpose:** It holds important information (called **state**) that needs to be shared across many different components. This includes things like:
    *   The currently logged-in user's details (`UserCredentials`).
    *   The list of all users in the system (`User[]`).
    *   Available institutional centers (`HIPCenter[]`).
    *   The user's collaborative projects (`HIPProject[]`).
    *   The currently selected project (`HIPProject`).
    *   A list of available software applications (`Application[]`).
    *   The user's running virtual desktops and apps (`Container[]`).
    *   Information related to [BIDS Dataset Handling](02_bids_dataset_handling_.md), like the selected dataset or participants.
*   **Central Hub:** It's a single, reliable place to find this shared data.
*   **Reactive:** When data in the AppStore changes (e.g., a new project is added), any component currently displaying that data automatically updates to show the latest information.

It ensures consistency and makes it easy for different parts of the application to access the data they need without complex wiring.

## Key Concepts

1.  **State:** Simply put, "state" is the data that an application needs to remember and track at any given moment to function correctly.
2.  **Global State:** This refers to state information that isn't just used by one component, but is needed by *multiple*, potentially unrelated, components across the application.
3.  **React Context (`AppContext`):** HIP uses a built-in React feature called "Context" to implement its global state. `AppContext` is like creating a shared "channel" or "whiteboard" that components can tune into.
4.  **Provider (`AppStoreProvider`):** This is a special React component that *provides* the global state to all the components inside it. It's the component that writes the shared information onto the `AppContext` whiteboard. In HIP, `AppStoreProvider` wraps almost the entire application, making the global state available everywhere.
5.  **Consumer (`useAppStore` Hook):** This is how individual components access the global state. It's a custom function (a React "Hook") that components can call to read data from the `AppContext` whiteboard or get functions to update that data.

## How It Works: Getting User Info Everywhere

Let's trace how Dr. Alice's user information and project list become available throughout the application after she logs in.

1.  **Application Start:** When the HIP application loads, the main `App` component is wrapped by the `AppStoreProvider` (as seen in [Chapter 5](05_frontend_application_core___routing_.md)'s `index.tsx`).
2.  **Initial Data Fetching:** The `AppStoreProvider` immediately gets to work. It uses functions from the [API Client Layer](07_api_client_layer_.md) to fetch essential initial data:
    *   It calls `getCurrentUser()` (from `src/nextcloudAuth.ts`) to get basic info about the logged-in user (like user ID).
    *   It calls `getUser()` to fetch more detailed user profile information.
    *   It calls `getProjectsForUser()` to get the list of projects Dr. Alice belongs to.
    *   It calls `getCenters()`, `getAvailableAppList()`, `getDesktopsAndApps()`, etc., to fetch other global information.
3.  **Storing in State:** As the data arrives from the API calls, the `AppStoreProvider` stores it in its internal state variables (managed using React's `useState` hook). For example, the user details go into a `user` state variable, and the project list goes into a `userProjects` state variable.
4.  **Providing the State:** The `AppStoreProvider` makes these state variables (like `user` and `userProjects`) and their update functions available through the `AppContext`.
5.  **Accessing the State (Sidebar):** Now, the `Sidebar` component needs to display Dr. Alice's projects. It calls the `useAppStore()` hook. This hook connects to the `AppContext` and gives the `Sidebar` access to the `userProjects` array stored by the `AppStoreProvider`. The Sidebar can then loop through this array and display the project names.
6.  **Accessing the State (Projects Page):** Similarly, the `Projects` component (which displays the main list of projects) also calls `useAppStore()` to get the same `userProjects` array.
7.  **Automatic Updates:** If Dr. Alice creates a *new* project later, the action would:
    *   Call the `createProject` API function.
    *   On success, the `AppStoreProvider` would be notified (or re-fetch the project list).
    *   The `AppStoreProvider` updates its internal `userProjects` state with the new list.
    *   **Magic!** Because both the `Sidebar` and the `Projects` page got the `userProjects` data via the `useAppStore()` hook, React automatically detects the change and re-renders *both* components to display the updated list, including the new project.

## Under the Hood: React Context and the `Store.tsx` File

HIP leverages React's Context API. This API is designed to share data that can be considered "global" for a tree of React components, like the current authenticated user or theme, without having to pass props down manually at every level (a problem called "prop drilling").

The core implementation lives in `src/Store.tsx`. Let's look at the key parts:

**1. Defining the State Shape (`IAppState` Interface):**
This defines *what* information will be stored globally.

```typescript
// Simplified from: src/Store.tsx

// Defines the structure of our global state "whiteboard"
export interface IAppState {
  // Tuple: [value, function to update value]
  user: [UserCredentials | null, React.Dispatch<React.SetStateAction<UserCredentials | null>>];
  users: [User[] | null, React.Dispatch<React.SetStateAction<User[] | null>>];
  centers: [HIPCenter[] | null, React.Dispatch<React.SetStateAction<HIPCenter[] | null>>];
  userProjects: [HIPProject[] | null, React.Dispatch<React.SetStateAction<HIPProject[] | null>>];
  selectedProject: [HIPProject | null, React.Dispatch<React.SetStateAction<HIPProject | null>>];
  availableApps: [Application[] | null, React.Dispatch<React.SetStateAction<Application[] | null>>];
  // User's private desktops/apps
  containers: [Container[] | null, React.Dispatch<React.SetStateAction<Container[] | null>>];
  // Desktops/apps for the selected project
  projectContainers: [Container[] | null, React.Dispatch<React.SetStateAction<Container[] | null>>];
  // ... other state like selectedBidsDataset, etc.
}
```

*   **Explanation:** This interface lists all the pieces of global information. Each entry is a "tuple" (a fixed-size array): the first element is the actual data (e.g., `userProjects` which is an array of `HIPProject` objects), and the second is the function React provides to *update* that data (e.g., `setUserProjects`).

**2. Creating the Context (`AppContext`):**
This creates the actual shared "channel" or "whiteboard".

```typescript
// Simplified from: src/Store.tsx
import React from 'react';
// ... other imports ...

// Create the actual Context object. Components will connect to this.
export const AppContext = React.createContext<IAppState>({} as IAppState);
```

*   **Explanation:** This line creates the `AppContext` using React's `createContext`. It's initially empty but expects the data to follow the `IAppState` structure.

**3. The Provider Component (`AppStoreProvider`):**
This component manages the state and makes it available.

```typescript
// Simplified from: src/Store.tsx
import React, { useState, useEffect, useMemo } from 'react';
import { getCurrentUser } from './nextcloudAuth';
import { getUser, getCenters, getProjectsForUser, /* ... other API calls ... */ } from './api/gatewayClientAPI';
// ... other imports like types and API functions ...

// This component wraps the app, fetches data, and provides the state.
export const AppStoreProvider = ({ children }: { children: JSX.Element }): JSX.Element => {
  // Use React's useState hook for each piece of global state
  const [user, setUser] = useState<UserCredentials | null>(null);
  const [userProjects, setUserProjects] = useState<HIPProject[] | null>(null);
  const [containers, setContainers] = useState<Container[] | null>(null);
  // ... other useState calls for centers, apps, etc. ...

  // useEffect hook runs once when the component mounts (app starts)
  useEffect(() => {
    const currentUser = getCurrentUser(); // Get basic user info
    if (!currentUser) return; // Stop if not logged in

    setUser(currentUser); // Store basic user info

    // Fetch detailed user info and update the user state
    getUser(currentUser.uid)
      .then(data => { if (data) setUser(prev => ({ ...prev, ...data })) })
      .catch(error => console.error("Error fetching user details:", error));

    // Fetch user's projects and store them
    getProjectsForUser(currentUser.uid || '')
      .then(projects => setUserProjects(projects))
      .catch(error => console.error("Error fetching projects:", error));

    // Fetch user's private desktops/apps and store them
    getDesktopsAndApps('private', currentUser.uid || '', [])
      .then(data => setContainers(data))
      .catch(error => console.error("Error fetching containers:", error));

    // ... other initial data fetching calls (centers, apps, etc.) ...

  }, []); // Empty dependency array means run only once on mount

  // Prepare the 'value' object to pass to the Provider
  // useMemo ensures this object is only recreated if state values change
  const value: IAppState = useMemo(() => ({
    user: [user, setUser],
    userProjects: [userProjects, setUserProjects],
    containers: [containers, setContainers],
    // ... include all other state variables and their setters ...
  }), [user, userProjects, containers /* ... other dependencies ... */]);

  // Return the Context Provider, wrapping the rest of the application ('children')
  // The 'value' object is now available to any component inside that uses useAppStore()
  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
};
```

*   **Explanation:**
    *   It uses `useState` to create state variables (`user`, `userProjects`, etc.) and their update functions (`setUser`, `setUserProjects`).
    *   It uses `useEffect` to run code *after* the component mounts. This is where the initial API calls are made to fetch data.
    *   The fetched data is stored in the state using the update functions (e.g., `setUserProjects(projects)`).
    *   It bundles the state variables and their update functions into a `value` object (matching the `IAppState` structure). `useMemo` optimizes this.
    *   Finally, it renders `AppContext.Provider`, passing the `value` object. Any component rendered within `{children}` can now access this `value`.

**4. The Consumer Hook (`useAppStore`):**
This is the simple function components use to tap into the context.

```typescript
// Simplified from: src/Store.tsx
import React, { useContext } from 'react';
// ... other imports ...

// Custom Hook to easily access the AppContext value in components
export const useAppStore = (): IAppState => {
  const context = useContext(AppContext); // Use React's useContext hook
  if (!context) {
    // This error occurs if useAppStore is used outside of AppStoreProvider
    throw new Error('useAppStore must be used within an AppStoreProvider');
  }
  return context; // Return the full state object
};
```

*   **Explanation:** This hook simply uses React's `useContext` hook to get the current `value` provided by the nearest `AppContext.Provider` and returns it.

**5. Using the Store in a Component:**
Here's how a component like the Sidebar might use it:

```typescript
// Simplified example: src/components/Sidebar.tsx
import React from 'react';
import { useAppStore } from '../Store'; // Import the hook
import { List, ListItem, ListItemText } from '@mui/material';

const Sidebar = () => {
  // Call the hook to get access to the global state
  const { userProjects } = useAppStore();
  // userProjects is the tuple: [projectsArray | null, setProjectsFunction]
  const [projects] = userProjects; // Extract just the array of projects

  return (
    <List>
      <ListItem>
        <ListItemText primary="My Projects" />
      </ListItem>
      {/* Map over the projects array from the global store */}
      {projects?.map(project => (
        <ListItem key={project.name}>
          {/* Display project title */}
          <ListItemText secondary={project.title} />
        </ListItem>
      ))}
      {/* ... other sidebar items ... */}
    </List>
  );
};

export default Sidebar;
```

*   **Explanation:** The `Sidebar` component calls `useAppStore()`. It then destructures the returned object to get `userProjects`. It takes the first element of the `userProjects` tuple (which is the actual array of projects) and uses it to render the list. If the `userProjects` array in the `AppStoreProvider` changes later, this component will automatically re-render.

**Conceptual Diagram:**

```mermaid
graph LR
    A[AppStoreProvider (Manages State in src/Store.tsx)] -- Provides Context --> B(AppContext);
    C[Sidebar Component] -- Calls useAppStore() --> D{useAppStore Hook};
    E[Projects Page Component] -- Calls useAppStore() --> D;
    F[Desktop Manager Component] -- Calls useAppStore() --> D;
    D -- Uses useContext() --> B;
    B -- Returns Current State --> D;
    D -- Provides State --> C;
    D -- Provides State --> E;
    D -- Provides State --> F;

    G[API Client Layer] -- Fetches Data --> A;
    A -- Stores Data --> A;
```

*   **Explanation:** The `AppStoreProvider` fetches data (using the API Client) and holds the state. It makes this state available via `AppContext`. Components use the `useAppStore` hook, which reads from `AppContext`, to get the latest shared state.

## Conclusion

You've learned about **Global State Management** in HIP, implemented using React Context as the **AppStore** (`AppContext`, `AppStoreProvider`, `useAppStore` hook). This system acts as a central hub for shared information like user details, projects, and available resources.

Key takeaways:

*   It solves the problem of sharing data efficiently and consistently across different components.
*   The `AppStoreProvider` fetches and holds the global state.
*   Components use the `useAppStore` hook to access this shared state.
*   Changes to the global state automatically trigger updates in components that use that state.

This central state management is crucial for keeping the different parts of the HIP application synchronized. The AppStore relies heavily on fetching data from the backend. In the next chapter, we'll take a closer look at how the frontend communicates with the backend server.

Next: [API Client Layer](07_api_client_layer_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)