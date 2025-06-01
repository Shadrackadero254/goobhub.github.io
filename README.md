# goobhub.github.io
import React, { useState, useEffect, createContext, useContext, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged, signOut, createUserWithEmailAndPassword, signInWithEmailAndPassword } from 'firebase/auth';
import { getFirestore, doc, getDoc, setDoc, collection, query, onSnapshot, addDoc, updateDoc, deleteDoc, where, getDocs, serverTimestamp } from 'firebase/firestore';

// --- Firebase Configuration & Context ---
// Global variables provided by the Canvas environment
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-goob-hub-app';

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

const AuthContext = createContext(null);
const FirestoreContext = createContext(null);

// --- Utility Functions ---
const generateUniqueId = () => crypto.randomUUID();

// --- Components ---

// Auth Provider to manage Firebase Auth state
const AuthProvider = ({ children }) => {
  const [currentUser, setCurrentUser] = useState(null);
  const [loadingAuth, setLoadingAuth] = useState(true);
  const [userId, setUserId] = useState(null);

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, async (user) => {
      if (user) {
        setCurrentUser(user);
        setUserId(user.uid);
      } else {
        // If no user, try to sign in with custom token or anonymously
        try {
          if (initialAuthToken) {
            await signInWithCustomToken(auth, initialAuthToken);
          } else {
            await signInAnonymously(auth);
          }
        } catch (error) {
          console.error("Firebase Auth Error:", error);
          // Fallback if anonymous sign-in also fails
          setUserId(crypto.randomUUID()); // Generate a random ID for unauthenticated users
        }
      }
      setLoadingAuth(false);
    });

    return () => unsubscribe();
  }, []);

  const login = async (email, password) => {
    try {
      await signInWithEmailAndPassword(auth, email, password);
      return { success: true };
    } catch (error) {
      console.error("Login error:", error);
      return { success: false, message: error.message };
    }
  };

  const signup = async (email, password) => {
    try {
      await createUserWithEmailAndPassword(auth, email, password);
      return { success: true };
    } catch (error) {
      console.error("Signup error:", error);
      return { success: false, message: error.message };
    }
  };

  const logout = async () => {
    try {
      await signOut(auth);
      // After logout, sign in anonymously to maintain a session for unauthenticated access
      await signInAnonymously(auth);
    } catch (error) {
      console.error("Logout error:", error);
    }
  };

  return (
    <AuthContext.Provider value={{ currentUser, userId, loadingAuth, login, signup, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

// Firestore Provider to manage data operations
const FirestoreProvider = ({ children }) => {
  const { userId, loadingAuth } = useContext(AuthContext);
  const [isFirestoreReady, setIsFirestoreReady] = useState(false);

  useEffect(() => {
    if (!loadingAuth && userId) {
      setIsFirestoreReady(true);
    } else {
      setIsFirestoreReady(false);
    }
  }, [loadingAuth, userId]);

  const getCollectionRef = (collectionName, isPublic = false, targetUserId = null) => { // Added targetUserId
    if (!isFirestoreReady) {
      console.error("Firestore is not ready. Auth or userId is missing.");
      return null;
    }
    // Use targetUserId if provided for specific user's private collection, otherwise current userId
    const baseId = targetUserId || userId;
    const basePath = isPublic ? `artifacts/${appId}/public/data` : `artifacts/${appId}/users/${baseId}`;
    return collection(db, `${basePath}/${collectionName}`);
  };

  const getDocRef = (collectionName, docId, isPublic = false, specificUserId = null) => { // Added specificUserId
    if (!isFirestoreReady) {
      console.error("Firestore is not ready. Auth or userId is missing.");
      return null;
    }
    // Use specificUserId if provided, otherwise use the current authenticated userId
    const targetUserId = specificUserId || userId;
    const basePath = isPublic ? `artifacts/${appId}/public/data` : `artifacts/${appId}/users/${targetUserId}`;
    return doc(db, `${basePath}/${collectionName}`, docId);
  };

  // Generic function to fetch all documents from a collection
  // Note: This function is primarily for fetching from a single known collection path.
  // For admin fetching across *all* users' private collections, a backend function is generally required.
  const fetchCollection = async (collectionName, isPublic = false, targetUserId = null) => {
    const colRef = getCollectionRef(collectionName, isPublic, targetUserId);
    if (!colRef) return [];
    try {
      const q = query(colRef);
      const snapshot = await getDocs(q);
      return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    } catch (e) {
      console.error(`Error fetching ${collectionName}:`, e);
      return [];
    }
  };

  // Generic function to add a document
  const addDocument = async (collectionName, data, isPublic = false) => {
    const colRef = getCollectionRef(collectionName, isPublic);
    if (!colRef) return null;
    try {
      const docRef = await addDoc(colRef, { ...data, createdAt: serverTimestamp() }); // Use serverTimestamp
      return docRef.id;
    } catch (e) {
      console.error(`Error adding document to ${collectionName}:`, e);
      return null;
    }
  };

  // Generic function to set/update a document by ID
  const setDocument = async (collectionName, docId, data, isPublic = false, specificUserId = null) => { // Added specificUserId
    const docRef = getDocRef(collectionName, docId, isPublic, specificUserId);
    if (!docRef) return false;
    try {
      await setDoc(docRef, { ...data, updatedAt: serverTimestamp() }, { merge: true }); // Use serverTimestamp
      return true;
    } catch (e) {
      console.error(`Error setting document in ${collectionName}:`, e);
      return false;
    }
  };

  // Generic function to delete a document
  const deleteDocument = async (collectionName, docId, isPublic = false, specificUserId = null) => { // Added specificUserId
    const docRef = getDocRef(collectionName, docId, isPublic, specificUserId);
    if (!docRef) return false;
    try {
      await deleteDoc(docRef);
      return true;
    } catch (e) {
      console.error(`Error deleting document from ${collectionName}:`, e);
      return false;
    }
  };

  return (
    <FirestoreContext.Provider value={{ db, isFirestoreReady, getCollectionRef, getDocRef, fetchCollection, addDocument, setDocument, deleteDocument }}>
      {children}
    </FirestoreContext.Provider>
  );
};

// --- UI Components ---

const LoadingSpinner = () => (
  <div className="flex items-center justify-center h-screen bg-gray-100">
    <div className="animate-spin rounded-full h-16 w-16 border-t-2 border-b-2 border-blue-500"></div>
    <p className="ml-4 text-lg text-gray-700">Loading GOOB HUB...</p>
  </div>
);

const AuthForm = ({ onLoginSuccess }) => {
  const [isLogin, setIsLogin] = useState(true);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [message, setMessage] = useState('');
  const { login, signup } = useContext(AuthContext);
  const [authResult, setAuthResult] = useState(null); // Store the result of login/signup

  const handleSubmit = async (e) => {
    e.preventDefault();
    setMessage('');
    setAuthResult(null); // Clear previous result
    let result;
    if (isLogin) {
      result = await login(email, password);
    } else {
      result = await signup(email, password);
    }
    setAuthResult(result); // Store the result
    if (result.success) {
      setMessage(`Successfully ${isLogin ? 'logged in' : 'signed up'}!`);
      onLoginSuccess();
    } else {
      setMessage(`Error: ${result.message}`);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
      <div className="bg-white p-8 rounded-xl shadow-2xl w-full max-w-md">
        <h2 className="text-3xl font-bold text-center text-gray-800 mb-6">
          {isLogin ? 'Login to GOOB HUB' : 'Sign Up for GOOB HUB'}
        </h2>
        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <label className="block text-gray-700 text-sm font-semibold mb-2" htmlFor="email">
              Email
            </label>
            <input
              type="email"
              id="email"
              className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-200"
              placeholder="your@email.com"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              required
            />
          </div>
          <div>
            <label className="block text-gray-700 text-sm font-semibold mb-2" htmlFor="password">
              Password
            </label>
            <input
              type="password"
              id="password"
              className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-200"
              placeholder="********"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              required
            />
          </div>
          <button
            type="submit"
            className="w-full bg-blue-600 text-white py-3 rounded-lg font-semibold text-lg hover:bg-blue-700 transition duration-300 shadow-md hover:shadow-lg"
          >
            {isLogin ? 'Login' : 'Sign Up'}
          </button>
        </form>
        {message && (
          <p className={`mt-4 text-center text-sm ${authResult?.success ? 'text-green-600' : 'text-red-600'}`}>
            {message}
          </p>
        )}
        <p className="mt-6 text-center text-gray-600 text-sm">
          {isLogin ? "Don't have an account?" : "Already have an account?"}{' '}
          <button
            onClick={() => setIsLogin(!isLogin)}
            className="text-blue-600 hover:underline font-semibold"
          >
            {isLogin ? 'Sign Up' : 'Login'}
          </button>
        </p>
      </div>
    </div>
  );
};

const Header = ({ userEmail, onLogout, userId, setView }) => {
  return (
    <header className="bg-white shadow-md p-4 flex justify-between items-center rounded-b-xl">
      <div className="flex items-center">
        <h1 className="text-2xl font-bold text-blue-700">GOOB HUB</h1>
        <span className="ml-4 text-sm text-gray-500">User ID: {userId}</span>
      </div>
      <nav className="flex items-center space-x-4">
        {userEmail && (
          <span className="text-gray-700 font-medium hidden sm:block">Welcome, {userEmail}</span>
        )}
        <button
          onClick={() => setView('user-notifications')} // New button for notifications
          className="px-3 py-1 bg-gray-200 text-gray-800 rounded-lg hover:bg-gray-300 transition duration-300 shadow-sm text-sm"
        >
          Notifications
        </button>
        <button
          onClick={onLogout}
          className="px-4 py-2 bg-red-500 text-white rounded-lg hover:bg-red-600 transition duration-300 shadow-sm"
        >
          Logout
        </button>
      </nav>
    </header>
  );
};

const Sidebar = ({ currentView, setView, userRole }) => {
  const navItems = {
    admin: [
      { name: 'Dashboard', key: 'admin-dashboard' },
      { name: 'Manage Users', key: 'admin-users' },
      { name: 'Platform Analytics', key: 'admin-analytics' },
      { name: 'Dispute Resolution', key: 'admin-disputes' },
      { name: 'Product Overview', key: 'admin-products-overview' }, // New Admin view
      { name: 'Settings', key: 'admin-settings' },
    ],
    retailer: [
      { name: 'Browse Products', key: 'retailer-products' },
      { name: 'My Orders', key: 'retailer-orders' },
      { name: 'Saved Lists', key: 'retailer-saved-lists' },
      { name: 'RFQ & Quotes', key: 'retailer-rfq' },
      { name: 'My Account', key: 'retailer-account' },
    ],
    wholesaler: [
      { name: 'My Products', key: 'wholesaler-products' },
      { name: 'Incoming Orders', key: 'wholesaler-orders' },
      { name: 'Performance Analytics', key: 'wholesaler-analytics' },
      { name: 'Manage Inventory', key: 'wholesaler-inventory' },
      { name: 'Manage Offers', key: 'wholesaler-offers' },
      { name: 'Incoming RFQs', key: 'wholesaler-rfqs' },
      { name: 'Distributor Orders', key: 'wholesaler-distributor-orders' },
    ],
  };

  return (
    <aside className="w-64 bg-gray-800 text-white p-6 rounded-r-xl shadow-lg">
      <h3 className="text-xl font-semibold mb-6 text-blue-300">Navigation</h3>
      <ul className="space-y-3">
        {navItems[userRole]?.map((item) => (
          <li key={item.key}>
            <button
              onClick={() => setView(item.key)}
              className={`w-full text-left px-4 py-2 rounded-lg transition duration-200 ${
                currentView === item.key
                  ? 'bg-blue-600 text-white shadow-md'
                  : 'hover:bg-gray-700 text-gray-200'
              }`}
            >
              {item.name}
            </button>
          </li>
        ))}
      </ul>
    </aside>
  );
};

// New Component: UserProfileModal (for Admin to view and 'edit' user profiles)
const UserProfileModal = ({ isOpen, onClose, userProfile, onUpdateProfile }) => {
  const [editingProfile, setEditingProfile] = useState(userProfile);
  const [message, setMessage] = useState('');

  useEffect(() => {
    setEditingProfile(userProfile); // Update local state when userProfile prop changes
  }, [userProfile]);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setEditingProfile(prev => ({ ...prev, [name]: value }));
  };

  const handleSave = async () => {
    setMessage('');
    // In a real system, this would call an admin backend function to update Firebase Auth
    // and the user's private profile. Here, we simulate by calling the passed prop function.
    const success = await onUpdateProfile(editingProfile.id, editingProfile);
    if (success) {
      setMessage('Profile updated (simulated) successfully!');
      setTimeout(onClose, 1500);
    } else {
      setMessage('Failed to update profile (simulated).');
    }
  };

  if (!isOpen || !userProfile) return null;

  return (
    <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
      <div className="bg-white rounded-xl shadow-2xl w-full max-w-md">
        <div className="p-4 border-b border-gray-200 flex justify-between items-center">
          <h3 className="text-xl font-semibold text-gray-800">Edit User Profile: {userProfile.email}</h3>
          <button onClick={onClose} className="text-gray-500 hover:text-gray-700 text-2xl">&times;</button>
        </div>
        <div className="p-4 space-y-3 text-gray-700">
          {message && (
            <div className={`bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg ${message.includes('Failed') ? 'bg-red-100 text-red-700' : ''}`} role="alert">
              {message}
            </div>
          )}
          <p><span className="font-semibold">User ID:</span> {editingProfile.id}</p>
          <p><span className="font-semibold">Email:</span> {editingProfile.email}</p>
          <div>
            <label className="block text-sm font-medium text-gray-700">Company Name</label>
            <input type="text" name="companyName" value={editingProfile.companyName || ''} onChange={handleChange} className="w-full p-2 border rounded-lg" />
          </div>
          <div>
            <label className="block text-sm font-medium text-gray-700">Contact Person</label>
            <input type="text" name="contactPerson" value={editingProfile.contactPerson || ''} onChange={handleChange} className="w-full p-2 border rounded-lg" />
          </div>
          <div>
            <label className="block text-sm font-medium text-gray-700">Phone</label>
            <input type="tel" name="phone" value={editingProfile.phone || ''} onChange={handleChange} className="w-full p-2 border rounded-lg" />
          </div>
          <div>
            <label className="block text-sm font-medium text-gray-700">Address</label>
            <input type="text" name="address" value={editingProfile.address || ''} onChange={handleChange} className="w-full p-2 border rounded-lg" />
          </div>
          <div>
            <label className="block text-sm font-medium text-gray-700">Role (Admin Assign)</label>
            <select name="role" value={editingProfile.role || ''} onChange={handleChange} className="w-full p-2 border rounded-lg">
              <option value="">Select Role</option>
              <option value="retailer">Retailer</option>
              <option value="distributor">Distributor</option>
              <option value="wholesaler">Wholesaler</option>
              <option value="admin">Admin</option>
            </select>
          </div>
          {/* Add more fields like status, permissions etc. */}
        </div>
        <div className="p-4 border-t border-gray-200 flex justify-end space-x-2">
          <button onClick={onClose} className="px-4 py-2 bg-gray-500 text-white rounded-lg hover:bg-gray-600">
            Cancel
          </button>
          <button onClick={handleSave} className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700">
            Save Changes (Simulated)
          </button>
        </div>
      </div>
    </div>
  );
};

const ManageUsers = () => {
  const { isFirestoreReady, db, setDocument } = useContext(FirestoreContext); // Added setDocument
  const [userProfiles, setUserProfiles] = useState([]);
  const [loading, setLoading] = useState(true);
  const [message, setMessage] = useState('');
  const [isProfileModalOpen, setIsProfileModalOpen] = useState(false);
  const [selectedUserProfile, setSelectedUserProfile] = useState(null);

  useEffect(() => {
    const fetchAllUserProfiles = async () => {
      if (!isFirestoreReady) {
        setMessage("Firestore is not ready.");
        setLoading(false);
        return;
      }
      setLoading(true);
      setMessage('');
      try {
        // IMPORTANT: In a real production system, securely listing *all* user profiles
        // (especially from different user's private collections) from the client-side
        // is NOT scalable or secure. This would typically require:
        // 1. Firebase Admin SDK on a secure backend (e.g., Cloud Functions)
        //    to list all users and their basic profile data.
        // 2. A separate, publicly readable (but admin-writable) collection for basic user metadata
        //    if needed for general admin overview, with strict security rules.
        // For this demo, we're simulating by trying to fetch profiles for a few hardcoded UIDs.
        // You would replace these with actual UIDs of users you create for testing.
        const mockUserIds = [
          'retailer_user_id_1', // Replace with actual UID of a retailer test user
          'wholesaler_user_id_1', // Replace with actual UID of a wholesaler test user
          'admin_user_id_1' // Replace with actual UID of an admin test user
        ];
        // For a more dynamic demo, you could try to query a public 'user_roles' collection
        // if you had one, to get all UIDs.

        const fetchedProfiles = [];
        for (const uid of mockUserIds) {
          try {
            const docRef = doc(db, `artifacts/${appId}/users/${uid}/userProfiles`, uid);
            const docSnap = await getDoc(docRef);
            if (docSnap.exists()) {
              fetchedProfiles.push({ id: uid, email: `${uid.substring(0, 8)}@example.com`, ...docSnap.data() });
            } else {
              fetchedProfiles.push({ id: uid, email: `${uid.substring(0, 8)}@example.com`, companyName: 'N/A', role: 'N/A', status: 'Active (No Profile)' });
            }
          } catch (error) {
            console.warn(`Could not fetch profile for user ${uid}:`, error);
            fetchedProfiles.push({ id: uid, email: `${uid.substring(0, 8)}@example.com`, companyName: 'N/A', role: 'N/A', status: 'Error Fetching (Permissions?)' });
          }
        }
        setUserProfiles(fetchedProfiles);
        setLoading(false);

      } catch (error) {
        console.error("Error fetching all user profiles (outer catch):", error);
        setMessage("Error loading user data. This feature requires specific admin rules/backend in a production system.");
        setLoading(false);
      }
    };

    fetchAllUserProfiles();
  }, [isFirestoreReady, db]);

  const openProfileModal = (profile) => {
    setSelectedUserProfile(profile);
    setIsProfileModalOpen(true);
  };

  const onUpdateUserProfile = async (userIdToUpdate, newProfileData) => {
    // This is a *simulated* update for the demo.
    // In a real system, this would involve calling a Firebase Cloud Function
    // (backend) with admin privileges to:
    // 1. Update the user's role in Firebase Authentication Custom Claims.
    // 2. Update their profile document in Firestore.
    try {
      // Simulate updating the user's profile in their private collection
      // This will only work if the admin's security rules allow this write.
      // E.g., `allow write: if request.auth.uid == 'YOUR_ADMIN_UID_HERE' || request.auth.token.admin == true;`
      const success = await setDocument('userProfiles', userIdToUpdate, newProfileData, false, userIdToUpdate);
      if (success) {
        // Update local state to reflect change (since Firestore listener might not pick up cross-user writes based on rules)
        setUserProfiles(prevProfiles =>
          prevProfiles.map(p => (p.id === userIdToUpdate ? { ...p, ...newProfileData } : p))
        );
        setMessage(`Simulated update for ${newProfileData.email} successful.`);
        return true;
      } else {
        setMessage(`Simulated update for ${newProfileData.email} failed.`);
        return false;
      }
    } catch (error) {
      console.error("Error simulating user profile update:", error);
      setMessage(`Simulated update for ${newProfileData.email} failed: ${error.message}`);
      return false;
    }
  };

  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">Manage Users</h2>
      <p className="text-gray-700 mb-4">
        As the Admin, you can view and manage all user accounts.
        <span className="font-semibold text-red-600"> (Note: This demo simulates fetching all users. A real production system requires Firebase Admin SDK for secure, scalable user listing.)</span>
      </p>
      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}
      <div className="bg-gray-50 p-4 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">User List</h3>
        <table className="min-w-full bg-white rounded-lg overflow-hidden">
          <thead className="bg-gray-200">
            <tr>
              <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">User ID</th>
              <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Email (Simulated)</th>
              <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Company Name</th>
              <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Role (Self-Declared)</th>
              <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Status</th>
              <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Actions</th>
            </tr>
          </thead>
          <tbody>
            {userProfiles.length > 0 ? (
              userProfiles.map(user => (
                <tr key={user.id} className="border-b border-gray-200 hover:bg-gray-100">
                  <td className="py-2 px-4 text-sm text-gray-700">{user.id}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{user.email}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{user.companyName}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{user.role}</td>
                  <td className="py-2 px-4 text-sm text-green-600">{user.status || 'Active'}</td>
                  <td className="py-2 px-4 text-sm">
                    <button onClick={() => openProfileModal(user)} className="text-blue-600 hover:underline mr-2">View/Edit Profile</button>
                    <button className="text-red-600 hover:underline">Deactivate (Simulated)</button>
                  </td>
                </tr>
              ))
            ) : (
              <tr>
                <td colSpan="6" className="py-4 text-center text-gray-600">No user profiles found. Sign up some users to populate this list.</td>
              </tr>
            )}
          </tbody>
        </table>
      </div>
      <div className="mt-6">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">Add New User (Admin Action)</h3>
        <form className="space-y-4">
          <input type="email" placeholder="Email" className="w-full p-2 border rounded-lg" />
          <input type="password" placeholder="Password" className="w-full p-2 border rounded-lg" />
          <select className="w-full p-2 border rounded-lg">
            <option>Retailer</option>
            <option>Wholesaler</option>
            <option>Distributor</option>
            <option>Admin</option>
          </select>
          <button className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700">Add User (Simulated)</button>
        </form>
      </div>
      {selectedUserProfile && (
        <UserProfileModal
          isOpen={isProfileModalOpen}
          onClose={closeProfileModal}
          userProfile={selectedUserProfile}
          onUpdateProfile={onUpdateUserProfile} // Pass the update function
        />
      )}
    </div>
  );
};

const ProductOverview = () => {
  const { isFirestoreReady, getCollectionRef } = useContext(FirestoreContext);
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [message, setMessage] = useState('');

  useEffect(() => {
    if (isFirestoreReady) {
      const productsColRef = getCollectionRef('products', true); // Fetch all products from public collection
      if (!productsColRef) {
        setMessage("Error: Could not get products collection reference.");
        setLoading(false);
        return;
      }
      const unsubscribe = onSnapshot(query(productsColRef), (snapshot) => {
        const fetchedProducts = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        setProducts(fetchedProducts);
        setLoading(false);
      }, (error) => {
        console.error("Error fetching all products for admin:", error);
        setMessage("Error loading all products. Check console for details.");
        setLoading(false);
      });
      return () => unsubscribe();
    }
  }, [isFirestoreReady, getCollectionRef]);

  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">Product Overview (Admin)</h2>
      <p className="text-gray-700 mb-4">
        View all products listed on the platform by all wholesalers and distributors.
      </p>

      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      {products.length === 0 ? (
        <p className="text-center text-gray-600">No products currently listed on the platform.</p>
      ) : (
        <div className="overflow-x-auto">
          <table className="min-w-full bg-white rounded-lg overflow-hidden shadow-sm">
            <thead className="bg-gray-200">
              <tr>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Product Name</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Supplier</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Category</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Price</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Stock</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Actions</th>
              </tr>
            </thead>
            <tbody>
              {products.map((product) => (
                <tr key={product.id} className="border-b border-gray-200 hover:bg-gray-100">
                  <td className="py-2 px-4 text-sm text-gray-700">{product.name}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{product.wholesalerName} ({product.wholesalerId?.substring(0,8)}...)</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{product.category}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">Ksh {product.price.toLocaleString()}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{product.stock}</td>
                  <td className="py-2 px-4 text-sm">
                    <button className="text-blue-600 hover:underline mr-2">View Details</button>
                    <button className="text-red-600 hover:underline">Remove</button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
};

const PlatformAnalytics = () => (
  <div className="p-6 bg-white rounded-xl shadow-md">
    <h2 className="text-3xl font-bold text-gray-800 mb-6">Platform Analytics</h2>
    <p className="text-gray-700 mb-4">
      Leverage data-driven insights to understand platform performance, user behavior,
      and market trends.
    </p>
    <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
      <div className="bg-purple-50 p-6 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-purple-700 mb-2">Sales Trends (Simulated Chart)</h3>
        <div className="h-48 bg-purple-100 flex items-center justify-center text-gray-600 rounded-lg">
          [Placeholder for Sales Volume Chart (e.g., Recharts)]
        </div>
        <p className="text-sm text-gray-600 mt-2">Monthly GMV, Average Order Value, Conversion Rates.</p>
      </div>
      <div className="bg-teal-50 p-6 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-teal-700 mb-2">Top Products & Categories (Simulated)</h3>
        <ul className="list-disc list-inside text-gray-700 space-y-1">
          <li>Product A (Electronics) - 1,500 units</li>
          <li>Product B (Groceries) - 1,200 units</li>
        </ul>
        <p className="text-sm text-gray-600 mt-2">Identify best-performing items and areas for growth.</p>
      </div>
      <div className="bg-orange-50 p-6 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-orange-700 mb-2">User Engagement (Simulated)</h3>
        <div className="h-48 bg-orange-100 flex items-center justify-center text-gray-600 rounded-lg">
          [Placeholder for User Activity Heatmap/Graph]
        </div>
        <p className="text-sm text-gray-600 mt-2">Active users, session duration, most visited pages.</p>
      </div>
      <div className="bg-pink-50 p-6 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-pink-700 mb-2">Supplier Performance (Simulated)</h3>
        <ul className="list-disc list-inside text-gray-700 space-y-1">
          <li>Supplier X: 98% Fulfillment Rate, Avg. Delivery 2 days</li>
          <li>Supplier Y: 92% Fulfillment Rate, Avg. Delivery 3 days</li>
        </ul>
        <p className="text-sm text-gray-600 mt-2">Monitor supplier reliability and efficiency.</p>
      </div>
    </div>
    <div className="mt-8 p-4 bg-gray-50 rounded-lg border border-gray-200">
      <h4 className="text-lg font-semibold text-gray-800 mb-2">Predictive Insights (AI Integration Placeholder)</h4>
      <p className="text-gray-700">
        Our AI engine can analyze historical data to forecast future demand, identify emerging trends,
        and suggest optimal pricing strategies.
      </p>
      <button className="mt-3 px-4 py-2 bg-purple-600 text-white rounded-lg hover:bg-purple-700">
        Generate Predictive Report
      </button>
    </div>
  </div>
);

const DisputeResolution = () => (
  <div className="p-6 bg-white rounded-xl shadow-md">
    <h2 className="text-3xl font-bold text-gray-800 mb-6">Dispute Resolution</h2>
    <p className="text-gray-700 mb-4">
      Manage and resolve disputes between retailers, distributors, and wholesalers efficiently.
      Our mediation tools help ensure fair outcomes.
    </p>
    <div className="bg-red-50 p-4 rounded-lg shadow-sm">
      <h3 className="text-xl font-semibold text-red-700 mb-3">Active Disputes (Simulated)</h3>
      <ul className="space-y-3">
        <li className="flex justify-between items-center bg-white p-3 rounded-lg shadow-sm border border-red-200">
          <div>
            <p className="font-semibold text-gray-800">#DISP-001: Order #GH-2025-00110 - Damaged Goods</p>
            <p className="text-sm text-gray-600">Retailer: "QuickMart" vs. Wholesaler: "AgroSupplies"</p>
            <p className="text-xs text-red-500">Status: Under Review - Priority</p>
          </div>
          <button className="px-3 py-1 bg-red-600 text-white text-sm rounded-lg hover:bg-red-700">View Details</button>
        </li>
        <li className="flex justify-between items-center bg-white p-3 rounded-lg shadow-sm border border-yellow-200">
          <div>
            <p className="font-semibold text-gray-800">#DISP-002: Order #GH-2025-00105 - Late Delivery</p>
            <p className="text-sm text-gray-600">Distributor: "TechDistro" vs. Logistics: "SwiftLogistics"</p>
            <p className="text-xs text-yellow-600">Status: Awaiting Response</p>
          </div>
          <button className="px-3 py-1 bg-yellow-600 text-white text-sm rounded-lg hover:bg-yellow-700">View Details</button>
        </li>
      </ul>
    </div>
    <div className="mt-6 p-4 bg-gray-50 rounded-lg border border-gray-200">
      <h4 className="text-lg font-semibold text-gray-800 mb-2">Dispute Resolution Process</h4>
      <p className="text-gray-700">
        Our system facilitates communication, evidence submission, and provides tools for mediation
        to reach a fair resolution.
      </p>
      <button className="mt-3 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700">
        View Resolution Guidelines
      </button>
    </div>
  </div>
);

// New Component: ProductReviewModal
const ProductReviewModal = ({ isOpen, onClose, productId, productName, wholesalerId, wholesalerName }) => {
  const { isFirestoreReady, addDocument } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [rating, setRating] = useState(0);
  const [reviewText, setReviewText] = useState('');
  const [message, setMessage] = useState('');

  const handleSubmitReview = async (e) => {
    e.preventDefault();
    if (!isFirestoreReady || !rating || !reviewText.trim()) {
      setMessage("Please provide a rating and a review.");
      return;
    }
    try {
      await addDocument('productReviews', {
        productId: productId,
        productName: productName,
        wholesalerId: wholesalerId,
        wholesalerName: wholesalerName,
        reviewerId: currentUser.uid,
        reviewerEmail: currentUser.email,
        rating: rating,
        reviewText: reviewText,
        reviewedAt: serverTimestamp(),
      }, true); // Product reviews are public
      setMessage('Review submitted successfully!');
      setRating(0);
      setReviewText('');
      setTimeout(onClose, 1500); // Close modal after a short delay
    } catch (error) {
      console.error("Error submitting review:", error);
      setMessage("Failed to submit review.");
    }
  };

  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
      <div className="bg-white rounded-xl shadow-2xl w-full max-w-md">
        <div className="p-4 border-b border-gray-200 flex justify-between items-center">
          <h3 className="text-xl font-semibold text-gray-800">Review {productName}</h3>
          <button onClick={onClose} className="text-gray-500 hover:text-gray-700 text-2xl">&times;</button>
        </div>
        <form onSubmit={handleSubmitReview} className="p-4 space-y-4">
          {message && (
            <div className={`bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg ${message.includes('Failed') ? 'bg-red-100 text-red-700' : ''}`} role="alert">
              {message}
            </div>
          )}
          <div>
            <label className="block text-gray-700 text-sm font-semibold mb-2">Rating</label>
            <div className="flex space-x-1">
              {[1, 2, 3, 4, 5].map((star) => (
                <span
                  key={star}
                  className={`cursor-pointer text-3xl ${star <= rating ? 'text-yellow-500' : 'text-gray-300'}`}
                  onClick={() => setRating(star)}
                >
                  &#9733;
                </span>
              ))}
            </div>
          </div>
          <div>
            <label className="block text-gray-700 text-sm font-semibold mb-2" htmlFor="reviewText">Your Review</label>
            <textarea
              id="reviewText"
              rows="4"
              value={reviewText}
              onChange={(e) => setReviewText(e.target.value)}
              className="w-full p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400"
              placeholder="Share your experience with this product..."
              required
            ></textarea>
          </div>
          <button
            type="submit"
            className="w-full py-2 bg-blue-600 text-white rounded-lg font-semibold hover:bg-blue-700 transition duration-300"
          >
            Submit Review
          </button>
        </form>
      </div>
    </div>
  );
};

// New Component: ProductDetailsModal
const ProductDetailsModal = ({ isOpen, onClose, product }) => {
  const { isFirestoreReady, getCollectionRef } = useContext(FirestoreContext);
  const [reviews, setReviews] = useState([]);
  const [loadingReviews, setLoadingReviews] = useState(true);

  useEffect(() => {
    if (isOpen && isFirestoreReady && product?.id) {
      const reviewsColRef = getCollectionRef('productReviews', true);
      if (reviewsColRef) {
        const q = query(reviewsColRef, where('productId', '==', product.id));
        const unsubscribe = onSnapshot(q, (snapshot) => {
          const fetchedReviews = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
          fetchedReviews.sort((a, b) => (b.reviewedAt?.toDate() || 0) - (a.reviewedAt?.toDate() || 0));
          setReviews(fetchedReviews);
          setLoadingReviews(false);
        }, (error) => {
          console.error("Error fetching product reviews:", error);
          setLoadingReviews(false);
        });
        return () => unsubscribe();
      }
    }
  }, [isOpen, isFirestoreReady, product?.id, getCollectionRef]);

  if (!isOpen || !product) return null;

  return (
    <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
      <div className="bg-white rounded-xl shadow-2xl w-full max-w-3xl h-5/6 flex flex-col">
        <div className="p-4 border-b border-gray-200 flex justify-between items-center">
          <h3 className="text-xl font-semibold text-gray-800">Product Details: {product.name}</h3>
          <button onClick={onClose} className="text-gray-500 hover:text-gray-700 text-2xl">&times;</button>
        </div>
        <div className="flex-1 p-4 overflow-y-auto">
          <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mb-6">
            {/* Product Info */}
            <div className="bg-gray-50 p-4 rounded-lg shadow-sm">
              <h4 className="text-lg font-semibold text-gray-800 mb-2">Product Information</h4>
              <p className="text-gray-700"><span className="font-semibold">Brand:</span> {product.brand}</p>
              <p className="text-gray-700"><span className="font-semibold">Category:</span> {product.category}</p>
              <p className="text-gray-700"><span className="font-semibold">Supplier:</span> {product.wholesalerName}</p>
              <p className="text-gray-700 mt-2">{product.description}</p>
              <p className="text-2xl font-bold text-green-700 mt-3">Ksh {product.price.toLocaleString()}</p>
              <p className="text-sm text-gray-600"><span className="font-semibold">Min. Order Quantity (MOQ):</span> {product.moq}</p>
              <p className="text-sm text-gray-600">
                <span className="font-semibold">Stock:</span> <span className={product.stock > 10 ? 'text-green-500' : 'text-orange-500'}>
                  {product.stock > 0 ? `${product.stock} units in stock` : 'Out of Stock'}
                </span>
              </p>
            </div>

            {/* Simulated AI Insights / Offers */}
            <div className="bg-blue-50 p-4 rounded-lg shadow-sm">
              <h4 className="text-lg font-semibold text-blue-700 mb-2">AI Insights & Offers</h4>
              <p className="text-gray-700">
                <span className="font-semibold">Popularity Trend:</span> <span className="text-green-600">Up 10% this month</span>
              </p>
              <p className="text-gray-700">
                <span className="font-semibold">Suggested Reorder Level:</span> {product.moq * 2} units
              </p>
              <p className="text-gray-700 mt-2">
                <span className="font-semibold">Current Offers:</span>
                <ul className="list-disc list-inside ml-4 text-sm">
                  <li>5% off for orders over {product.moq * 5} units</li>
                  <li>Free delivery for orders over Ksh 10,000</li>
                </ul>
              </p>
            </div>
          </div>

          {/* Product Reviews Section */}
          <div className="mt-6">
            <h4 className="text-xl font-semibold text-gray-800 mb-3">Customer Reviews</h4>
            {loadingReviews ? (
              <p className="text-gray-600">Loading reviews...</p>
            ) : reviews.length === 0 ? (
              <p className="text-gray-600">No reviews yet for this product.</p>
            ) : (
              <div className="space-y-4">
                {reviews.map(review => (
                  <div key={review.id} className="bg-gray-100 p-3 rounded-lg shadow-sm border border-gray-200">
                    <div className="flex items-center mb-1">
                      <span className="font-semibold text-yellow-500 text-lg mr-2">
                        {'★'.repeat(review.rating)}{'☆'.repeat(5 - review.rating)}
                      </span>
                      <span className="text-sm text-gray-700">{review.reviewerEmail?.split('@')[0]}</span>
                      <span className="text-xs text-gray-500 ml-auto">
                        {review.reviewedAt ? new Date(review.reviewedAt.toDate()).toLocaleDateString() : 'N/A'}
                      </span>
                    </div>
                    <p className="text-gray-800">{review.reviewText}</p>
                  </div>
                ))}
              </div>
            )}
          </div>
        </div>
        <div className="p-4 border-t border-gray-200 flex justify-end">
          <button onClick={onClose} className="px-4 py-2 bg-gray-500 text-white rounded-lg hover:bg-gray-600">
            Close
          </button>
        </div>
      </div>
    </div>
  );
};


const RetailerProducts = () => {
  const { addDocument, isFirestoreReady, getCollectionRef, setDocument, getDocRef } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [searchTerm, setSearchTerm] = useState('');
  const [selectedCategory, setSelectedCategory] = useState('All');
  const [minPrice, setMinPrice] = useState('');
  const [maxPrice, setMaxPrice] = useState('');
  const [moqFilter, setMoqFilter] = useState('');
  const [message, setMessage] = useState('');
  const [savedLists, setSavedLists] = useState([]);
  const [selectedListId, setSelectedListId] = useState('');
  const [isReviewModalOpen, setIsReviewModalOpen] = useState(false);
  const [selectedProductForReview, setSelectedProductForReview] = useState(null);
  const [isProductDetailsModalOpen, setIsProductDetailsModalOpen] = useState(false); // New state
  const [selectedProductForDetails, setSelectedProductForDetails] = useState(null); // New state

  useEffect(() => {
    if (isFirestoreReady) {
      const productsColRef = getCollectionRef('products', true);
      if (!productsColRef) {
        setMessage("Error: Could not get products collection reference.");
        setLoading(false);
        return;
      }
      const unsubscribeProducts = onSnapshot(query(productsColRef), (snapshot) => {
        const fetchedProducts = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        setProducts(fetchedProducts);
        setLoading(false);
      }, (error) => {
        console.error("Error fetching products:", error);
        setMessage("Error loading products. Check console for details.");
        setLoading(false);
      });

      // Fetch saved lists for the current user
      if (currentUser) {
        const listsColRef = getCollectionRef('savedLists', false);
        if (listsColRef) {
          const unsubscribeLists = onSnapshot(query(listsColRef), (snapshot) => {
            setSavedLists(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
          }, (error) => {
            console.error("Error fetching saved lists:", error);
          });
          return () => { unsubscribeProducts(); unsubscribeLists(); };
        }
      }
      return () => unsubscribeProducts();
    }
  }, [isFirestoreReady, getCollectionRef, currentUser]);

  const handleAddToCart = async (product) => {
    if (!isFirestoreReady) {
      setMessage("Please wait, system is initializing.");
      return;
    }
    try {
      const orderData = {
        productId: product.id,
        productName: product.name,
        quantity: 1, // For simplicity, assume 1 unit for now, can be updated
        price: product.price,
        wholesalerId: product.wholesalerId,
        wholesalerName: product.wholesalerName,
        status: 'Pending',
        orderDate: serverTimestamp(), // Use serverTimestamp
        retailerId: auth.currentUser?.uid,
        retailerEmail: auth.currentUser?.email,
      };
      await addDocument('orders', orderData, false); // Retailer's private order collection
      setMessage(`Added ${product.name} to cart!`);
    } catch (error) {
      console.error("Error adding to cart:", error);
      setMessage("Failed to add to cart.");
    }
  };

  const handleAddToList = async (productId) => {
    if (!isFirestoreReady || !selectedListId) {
      setMessage("Please select a list first.");
      return;
    }
    try {
      const listDocRef = getDocRef('savedLists', selectedListId, false);
      const docSnap = await getDoc(listDocRef);
      if (docSnap.exists()) {
        const currentProducts = docSnap.data().products || [];
        if (!currentProducts.includes(productId)) {
          await setDocument('savedLists', selectedListId, { products: [...currentProducts, productId] }, false);
          setMessage(`Product added to "${savedLists.find(l => l.id === selectedListId)?.name}"!`);
        } else {
          setMessage('Product already in this list.');
        }
      } else {
        setMessage('Selected list not found.');
      }
    } catch (error) {
      console.error("Error adding to list:", error);
      setMessage("Failed to add product to list.");
    }
  };

  const openReviewModal = (product) => {
    setSelectedProductForReview(product);
    setIsReviewModalOpen(true);
  };

  const closeReviewModal = () => {
    setIsReviewModalOpen(false);
    setSelectedProductForReview(null);
  };

  const openProductDetailsModal = (product) => { // New function
    setSelectedProductForDetails(product);
    setIsProductDetailsModalOpen(true);
  };

  const closeProductDetailsModal = () => { // New function
    setIsProductDetailsModalOpen(false);
    setSelectedProductForDetails(null);
  };


  const filteredProducts = products.filter(product => {
    const matchesSearch = searchTerm ? (
      product.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
      product.description.toLowerCase().includes(searchTerm.toLowerCase()) ||
      product.brand.toLowerCase().includes(searchTerm.toLowerCase())
    ) : true;
    const matchesCategory = selectedCategory === 'All' || product.category === selectedCategory;
    const matchesPrice = (!minPrice || product.price >= parseFloat(minPrice)) &&
                         (!maxPrice || product.price <= parseFloat(maxPrice));
    const matchesMoq = (!moqFilter || product.moq <= parseInt(moqFilter)); // Assuming filter is max MOQ

    return matchesSearch && matchesCategory && matchesPrice && matchesMoq;
  });

  const categories = ['All', ...new Set(products.map(p => p.category))];

  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">Browse Products</h2>
      <p className="text-gray-700 mb-4">
        Explore a wide range of products from various wholesalers and distributors.
      </p>

      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      {/* Advanced Search & Filtering */}
      <div className="bg-gray-50 p-4 rounded-lg shadow-sm mb-6 grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        <div>
          <label htmlFor="search" className="block text-sm font-medium text-gray-700">Search</label>
          <input
            type="text"
            id="search"
            placeholder="Product name, brand, SKU..."
            className="w-full p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400"
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
          />
        </div>
        <div>
          <label htmlFor="category" className="block text-sm font-medium text-gray-700">Category</label>
          <select
            id="category"
            className="w-full p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400"
            value={selectedCategory}
            onChange={(e) => setSelectedCategory(e.target.value)}
          >
            {categories.map(cat => (
              <option key={cat} value={cat}>{cat}</option>
            ))}
          </select>
        </div>
        <div>
          <label htmlFor="minPrice" className="block text-sm font-medium text-gray-700">Min Price</label>
          <input
            type="number"
            id="minPrice"
            placeholder="Min Price"
            className="w-full p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400"
            value={minPrice}
            onChange={(e) => setMinPrice(e.target.value)}
          />
        </div>
        <div>
          <label htmlFor="maxPrice" className="block text-sm font-medium text-gray-700">Max Price</label>
          <input
            type="number"
            id="maxPrice"
            placeholder="Max Price"
            className="w-full p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400"
            value={maxPrice}
            onChange={(e) => setMaxPrice(e.target.value)}
          />
        </div>
        <div>
          <label htmlFor="moqFilter" className="block text-sm font-medium text-gray-700">Max MOQ</label>
          <input
            type="number"
            id="moqFilter"
            placeholder="Max MOQ"
            className="w-full p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400"
            value={moqFilter}
            onChange={(e) => setMoqFilter(e.target.value)}
          />
        </div>
      </div>

      {/* Product Recommendations (AI Placeholder) */}
      <div className="bg-blue-50 p-4 rounded-lg shadow-sm mb-6 border border-blue-200">
        <h3 className="text-xl font-semibold text-blue-700 mb-3">AI Powered Recommendations</h3>
        <p className="text-gray-700">
          Based on your Browse history and similar businesses, you might be interested in:
        </p>
        <div className="flex flex-wrap gap-3 mt-3">
          <span className="bg-blue-200 text-blue-800 text-sm px-3 py-1 rounded-full">Bulk Sugar</span>
          <span className="bg-blue-200 text-blue-800 text-sm px-3 py-1 rounded-full">Cooking Oil 20L</span>
          <span className="bg-blue-200 text-blue-800 text-sm px-3 py-1 rounded-full">Washing Detergent 5kg</span>
        </div>
      </div>

      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
        {filteredProducts.length > 0 ? (
          filteredProducts.map((product) => (
            <div key={product.id} className="bg-gray-50 p-6 rounded-lg shadow-md border border-gray-200 hover:shadow-lg transition duration-200">
              <h3 className="text-xl font-bold text-gray-800 mb-2">{product.name}</h3>
              <p className="text-sm text-gray-600 mb-1">Brand: {product.brand}</p>
              <p className="text-sm text-gray-600 mb-1">Category: {product.category}</p>
              <p className="text-sm text-gray-600 mb-2">Supplier: {product.wholesalerName}</p>
              <p className="text-gray-700 mb-3">{product.description}</p>
              <p className="text-2xl font-bold text-green-700 mb-3">Ksh {product.price.toLocaleString()}</p>
              <p className="text-sm text-gray-600 mb-3">Min. Order Quantity (MOQ): {product.moq}</p>
              <p className="text-sm text-gray-600 mb-3">
                Stock: <span className={product.stock > 10 ? 'text-green-500' : 'text-orange-500'}>
                  {product.stock > 0 ? `${product.stock} units in stock` : 'Out of Stock'}
                </span>
              </p>
              <button
                onClick={() => handleAddToCart(product)}
                disabled={product.stock === 0}
                className={`w-full py-2 rounded-lg font-semibold transition duration-300 ${
                  product.stock === 0
                    ? 'bg-gray-400 text-gray-700 cursor-not-allowed'
                    : 'bg-blue-600 text-white hover:bg-blue-700 shadow-md'
                }`}
              >
                {product.stock === 0 ? 'Out of Stock' : 'Add to Cart'}
              </button>
              <div className="mt-2 flex items-center space-x-2">
                <select
                  value={selectedListId}
                  onChange={(e) => setSelectedListId(e.target.value)}
                  className="flex-grow p-2 border border-gray-300 rounded-lg text-sm"
                >
                  <option value="">Add to List...</option>
                  {savedLists.map(list => (
                    <option key={list.id} value={list.id}>{list.name}</option>
                  ))}
                </select>
                <button
                  onClick={() => handleAddToList(product.id)}
                  disabled={!selectedListId}
                  className="px-3 py-2 bg-purple-600 text-white rounded-lg text-sm hover:bg-purple-700 transition duration-300"
                >
                  Add
                </button>
              </div>
              <button className="w-full mt-2 py-2 rounded-lg font-semibold bg-indigo-500 text-white hover:bg-indigo-600 transition duration-300 shadow-md">
                Request for Quote (RFQ)
              </button>
              <button
                onClick={() => openReviewModal(product)}
                className="w-full mt-2 py-2 rounded-lg font-semibold bg-yellow-500 text-white hover:bg-yellow-600 transition duration-300 shadow-md"
              >
                Leave Review
              </button>
              <button
                onClick={() => openProductDetailsModal(product)} // New button to open product details
                className="w-full mt-2 py-2 rounded-lg font-semibold bg-gray-700 text-white hover:bg-gray-800 transition duration-300 shadow-md"
              >
                View Details
              </button>
            </div>
          ))
        ) : (
          <p className="text-center text-gray-600 col-span-full">No products found matching your criteria.</p>
        )}
      </div>
      {selectedProductForReview && (
        <ProductReviewModal
          isOpen={isReviewModalOpen}
          onClose={closeReviewModal}
          productId={selectedProductForReview.id}
          productName={selectedProductForReview.name}
          wholesalerId={selectedProductForReview.wholesalerId}
          wholesalerName={selectedProductForReview.wholesalerName}
        />
      )}
      {selectedProductForDetails && ( // New modal for product details
        <ProductDetailsModal
          isOpen={isProductDetailsModalOpen}
          onClose={closeProductDetailsModal}
          product={selectedProductForDetails}
        />
      )}
    </div>
  );
};

const MessagingModal = ({ isOpen, onClose, orderId, participant1Id, participant2Id, participant1Email, participant2Email }) => {
  const { isFirestoreReady, addDocument, getCollectionRef } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [messages, setMessages] = useState([]);
  const [newMessage, setNewMessage] = useState('');
  const messagesEndRef = useRef(null);

  const conversationId = [participant1Id, participant2Id].sort().join('_'); // Consistent ID for conversation

  useEffect(() => {
    if (isOpen && isFirestoreReady) {
      const q = query(
        getCollectionRef('messages', true), // Messages are public for demo simplicity
        where('conversationId', '==', conversationId),
        // orderBy('timestamp') // Ordering requires an index in Firestore
      );
      const unsubscribe = onSnapshot(q, (snapshot) => {
        const fetchedMessages = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        // Sort in client-side for now, as orderBy requires index
        fetchedMessages.sort((a, b) => (a.timestamp?.toDate() || 0) - (b.timestamp?.toDate() || 0));
        setMessages(fetchedMessages);
      }, (error) => {
        console.error("Error fetching messages:", error);
      });
      return () => unsubscribe();
    }
  }, [isOpen, isFirestoreReady, conversationId, getCollectionRef]);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  const handleSendMessage = async (e) => {
    e.preventDefault();
    if (!newMessage.trim() || !isFirestoreReady) return;

    try {
      await addDocument('messages', {
        conversationId: conversationId,
        orderId: orderId,
        senderId: currentUser.uid,
        senderEmail: currentUser.email,
        receiverId: currentUser.uid === participant1Id ? participant2Id : participant1Id,
        receiverEmail: currentUser.uid === participant1Id ? participant2Email : participant1Email,
        content: newMessage,
        timestamp: serverTimestamp(),
      }, true); // Messages are public for demo simplicity
      setNewMessage('');
    } catch (error) {
      console.error("Error sending message:", error);
    }
  };

  if (!isOpen) return null;

  return (
    <div className="w-full h-full flex flex-col border border-gray-300 rounded-lg">
      <div className="flex-1 p-4 overflow-y-auto space-y-3 bg-gray-50">
        {messages.length === 0 ? (
          <p className="text-center text-gray-500">No messages yet. Start the conversation!</p>
        ) : (
          messages.map((msg) => (
            <div
              key={msg.id}
              className={`flex ${msg.senderId === currentUser.uid ? 'justify-end' : 'justify-start'}`}
            >
              <div
                className={`max-w-[70%] p-3 rounded-lg shadow-sm ${
                  msg.senderId === currentUser.uid
                    ? 'bg-blue-500 text-white rounded-br-none'
                    : 'bg-gray-200 text-gray-800 rounded-bl-none'
                }`}
              >
                <p className="font-semibold text-sm mb-1">
                  {msg.senderId === currentUser.uid ? 'You' : msg.senderEmail?.split('@')[0]}
                </p>
                <p>{msg.content}</p>
                {msg.timestamp && (
                  <p className="text-xs mt-1 opacity-80">
                    {new Date(msg.timestamp.toDate()).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
                  </p>
                )}
              </div>
            </div>
          ))
        )}
        <div ref={messagesEndRef} />
      </div>
      <form onSubmit={handleSendMessage} className="p-3 border-t border-gray-200 flex space-x-2 bg-white">
        <input
          type="text"
          value={newMessage}
          onChange={(e) => setNewMessage(e.target.value)}
          placeholder="Type your message..."
          className="flex-1 p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400"
        />
        <button
          type="submit"
          className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition duration-300"
        >
          Send
        </button>
      </form>
    </div>
  );
};


const MyOrders = ({ role }) => {
  const { getCollectionRef, isFirestoreReady, setDocument } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [orders, setOrders] = useState([]);
  const [loading, setLoading] = useState(true);
  const [message, setMessage] = useState('');
  const [isOrderDetailsModalOpen, setIsOrderDetailsModalOpen] = useState(false);
  const [selectedOrderDetails, setSelectedOrderDetails] = useState(null);

  useEffect(() => {
    if (isFirestoreReady && currentUser) {
      let q;
      // Orders are stored in the user's private space
      q = query(getCollectionRef('orders', false), 
                role === 'retailer' ? where('retailerId', '==', currentUser.uid) : where('wholesalerId', '==', currentUser.uid));

      const unsubscribe = onSnapshot(q, (snapshot) => {
        const fetchedOrders = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        // Sort orders by date, newest first
        fetchedOrders.sort((a, b) => (b.orderDate?.toDate() || 0) - (a.orderDate?.toDate() || 0)); // Use toDate() for serverTimestamp
        setOrders(fetchedOrders);
        setLoading(false);
      }, (error) => {
        console.error("Error fetching orders:", error);
        setMessage("Error loading orders.");
        setLoading(false);
      });
      return () => unsubscribe();
    }
  }, [isFirestoreReady, currentUser, role, getCollectionRef]);

  const updateOrderStatus = async (orderId, newStatus, currentRetailerId, currentWholesalerId) => {
    if (!isFirestoreReady) {
      setMessage("Please wait, system is initializing.");
      return;
    }
    try {
      const trackingNumber = newStatus === 'Shipped' ? `GH-TRK-${generateUniqueId().substring(0, 8).toUpperCase()}` : null;

      // Update the order in the retailer's space
      if (currentRetailerId) {
        const retailerOrderDocRef = doc(db, `artifacts/${appId}/users/${currentRetailerId}/orders`, orderId);
        await setDoc(retailerOrderDocRef, { status: newStatus, trackingNumber: trackingNumber, updatedAt: serverTimestamp() }, { merge: true });
      }
      // Update the order in the wholesaler's space (if different)
      if (currentWholesalerId && currentWholesalerId !== currentRetailerId) {
        const wholesalerOrderDocRef = doc(db, `artifacts/${appId}/users/${currentWholesalerId}/orders`, orderId);
        await setDoc(wholesalerOrderDocRef, { status: newStatus, trackingNumber: trackingNumber, updatedAt: serverTimestamp() }, { merge: true });
      }

      setMessage(`Order ${orderId} status updated to ${newStatus}.`);
    } catch (error) {
      console.error("Error updating order status:", error);
      setMessage("Failed to update order status.");
    }
  };

  const handleMakePayment = (orderId) => {
    setMessage(`Simulating payment for Order #${orderId}... Payment successful!`);
    // In a real application, this would integrate with a payment gateway.
    // After successful payment, you'd update the order status to 'Paid' or similar.
    // For demo, we just show a message.
  };

  const openOrderDetailsModal = (order) => {
    setSelectedOrderDetails(order);
    setIsOrderDetailsModalOpen(true);
  };

  const closeOrderDetailsModal = () => {
    setIsOrderDetailsModalOpen(false);
    setSelectedOrderDetails(null);
  };


  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">{role === 'retailer' ? 'My Orders' : 'Incoming Orders'}</h2>
      <p className="text-gray-700 mb-4">
        {role === 'retailer'
          ? 'Track the status of your bulk purchases.'
          : 'View and manage orders placed by retailers and distributors.'}
      </p>

      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      {orders.length === 0 ? (
        <p className="text-center text-gray-600">No orders found.</p>
      ) : (
        <div className="space-y-4">
          {orders.map((order) => (
            <div key={order.id} className="bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
              <div className="flex justify-between items-center mb-2">
                <h3 className="text-lg font-semibold text-gray-800">Order #{order.id}</h3>
                <span className={`px-3 py-1 rounded-full text-sm font-medium ${
                  order.status === 'Pending' ? 'bg-yellow-200 text-yellow-800' :
                  order.status === 'Shipped' ? 'bg-blue-200 text-blue-800' :
                  order.status === 'Delivered' ? 'bg-green-200 text-green-800' :
                  'bg-gray-200 text-gray-800'
                }`}>
                  {order.status}
                </span>
              </div>
              <p className="text-gray-700">Product: {order.productName}</p>
              <p className="text-gray-700">Quantity: {order.quantity}</p>
              <p className="text-gray-700">Total Price: Ksh {(order.price * order.quantity).toLocaleString()}</p>
              <p className="text-sm text-gray-600">Order Date: {order.orderDate ? new Date(order.orderDate.toDate()).toLocaleDateString() : 'N/A'}</p>
              {role === 'retailer' && <p className="text-sm text-gray-600">Supplier: {order.wholesalerName || order.wholesalerId}</p>}
              {role === 'wholesaler' && <p className="text-sm text-gray-600">Ordered by: {order.retailerEmail || order.retailerId}</p>}
              
              {order.trackingNumber && (
                <p className="text-sm text-gray-700 mt-2">
                  Tracking #: <a href={`https://example.com/track?id=${order.trackingNumber}`} target="_blank" rel="noopener noreferrer" className="text-blue-600 hover:underline">
                    {order.trackingNumber}
                  </a>
                </p>
              )}


              {role === 'wholesaler' && order.status === 'Pending' && (
                <div className="mt-3 flex space-x-2">
                  <button
                    onClick={() => updateOrderStatus(order.id, 'Shipped', order.retailerId, order.wholesalerId)}
                    className="px-3 py-1 bg-green-600 text-white rounded-lg text-sm hover:bg-green-700"
                  >
                    Mark as Shipped
                  </button>
                  <button
                    onClick={() => updateOrderStatus(order.id, 'Cancelled', order.retailerId, order.wholesalerId)}
                    className="px-3 py-1 bg-red-600 text-white rounded-lg text-sm hover:bg-red-700"
                  >
                    Cancel Order
                  </button>
                </div>
              )}
              {role === 'retailer' && order.status === 'Shipped' && (
                <div className="mt-3 flex space-x-2">
                  <button
                    onClick={() => updateOrderStatus(order.id, 'Delivered', order.retailerId, order.wholesalerId)}
                    className="px-3 py-1 bg-green-600 text-white rounded-lg text-sm hover:bg-green-700"
                  >
                    Mark as Delivered
                  </button>
                  <button className="px-3 py-1 bg-indigo-600 text-white rounded-lg text-sm hover:bg-indigo-700">
                    Track Shipment (Simulated)
                  </button>
                </div>
              )}
              {role === 'retailer' && order.status !== 'Delivered' && ( // Simulate payment option for retailer
                <div className="mt-3">
                  <button
                    onClick={() => handleMakePayment(order.id)}
                    className="px-3 py-1 bg-purple-600 text-white rounded-lg text-sm hover:bg-purple-700"
                  >
                    Make Payment (Simulated)
                  </button>
                </div>
              )}
              <button
                onClick={() => openOrderDetailsModal(order)}
                className="mt-3 ml-2 px-3 py-1 bg-gray-600 text-white rounded-lg text-sm hover:bg-gray-700"
              >
                View Details & Chat
              </button>
              {/* Logistics Integration Placeholder */}
              <div className="mt-3 p-2 bg-gray-100 rounded-lg text-sm text-gray-600">
                <p className="font-semibold">Logistics Status (Simulated):</p>
                <p>Current: {order.status === 'Shipped' ? 'In Transit, Nairobi Depot' : 'Awaiting Shipment'}</p>
                <p>Estimated Delivery: {order.status === 'Shipped' ? new Date(new Date().setDate(new Date().getDate() + 2)).toLocaleDateString() : 'N/A'}</p>
              </div>
            </div>
          ))}
        </div>
      )}
      {selectedOrderDetails && (
        <OrderDetailsModal
          isOpen={isOrderDetailsModalOpen}
          onClose={closeOrderDetailsModal}
          order={selectedOrderDetails}
        />
      )}
    </div>
  );
};

const SavedLists = () => {
  const { isFirestoreReady, getCollectionRef, addDocument, setDocument, deleteDocument } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [lists, setLists] = useState([]);
  const [loading, setLoading] = useState(true);
  const [message, setMessage] = useState('');
  const [newListName, setNewListName] = useState('');
  const [products, setProducts] = useState([]); // To display product names in lists

  useEffect(() => {
    if (isFirestoreReady && currentUser) {
      const listsColRef = getCollectionRef('savedLists', false);
      if (listsColRef) {
        const unsubscribeLists = onSnapshot(query(listsColRef), (snapshot) => {
          setLists(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
          setLoading(false);
        }, (error) => {
          console.error("Error fetching saved lists:", error);
          setMessage("Error loading saved lists.");
          setLoading(false);
        });
        return () => unsubscribeLists();
      } else {
        setLoading(false);
      }
    }
  }, [isFirestoreReady, currentUser, getCollectionRef]);

  // Fetch all products to display names in saved lists
  useEffect(() => {
    if (isFirestoreReady) {
      const productsColRef = getCollectionRef('products', true);
      if (productsColRef) {
        const unsubscribeProducts = onSnapshot(query(productsColRef), (snapshot) => {
          setProducts(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        });
        return () => unsubscribeProducts();
      }
    }
  }, [isFirestoreReady, getCollectionRef]);

  const handleCreateList = async (e) => {
    e.preventDefault();
    if (!newListName.trim() || !isFirestoreReady) {
      setMessage("List name cannot be empty.");
      return;
    }
    try {
      await addDocument('savedLists', { name: newListName, products: [] }, false);
      setMessage(`List "${newListName}" created successfully!`);
      setNewListName('');
    }
    catch (error) {
      console.error("Error creating list:", error);
      setMessage("Failed to create list.");
    }
  };

  const handleRemoveProductFromList = async (listId, productIdToRemove) => {
    if (!isFirestoreReady) return;
    try {
      const listToUpdate = lists.find(list => list.id === listId);
      if (listToUpdate) {
        const updatedProducts = listToUpdate.products.filter(id => id !== productIdToRemove);
        await setDocument('savedLists', listId, { products: updatedProducts }, false);
        setMessage('Product removed from list.');
      }
    } catch (error) {
      console.error("Error removing product from list:", error);
      setMessage("Failed to remove product from list.");
    }
  };

  const handleDeleteList = async (listId) => {
    if (!isFirestoreReady) return;
    if (window.confirm('Are you sure you want to delete this list?')) {
      try {
        await deleteDocument('savedLists', listId, false);
        setMessage('List deleted successfully!');
      } catch (error) {
        console.error("Error deleting list:", error);
        setMessage("Failed to delete list.");
      }
    }
  };

  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">My Saved Lists</h2>
      <p className="text-gray-700 mb-4">
        Create and manage custom shopping lists for easy reordering or future reference.
      </p>

      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      <div className="mb-8 bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">Create New List</h3>
        <form onSubmit={handleCreateList} className="flex space-x-2">
          <input
            type="text"
            placeholder="New List Name"
            value={newListName}
            onChange={(e) => setNewListName(e.target.value)}
            className="flex-grow p-2 border rounded-lg"
            required
          />
          <button
            type="submit"
            className="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700 shadow-md"
          >
            Create List
          </button>
        </form>
      </div>

      {lists.length === 0 ? (
        <p className="text-center text-gray-600">You have no saved lists yet. Create one above!</p>
      ) : (
        <div className="space-y-6">
          {lists.map((list) => (
            <div key={list.id} className="bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
              <div className="flex justify-between items-center mb-3">
                <h3 className="text-xl font-semibold text-gray-800">{list.name}</h3>
                <button
                  onClick={() => handleDeleteList(list.id)}
                  className="px-3 py-1 bg-red-500 text-white rounded-lg text-sm hover:bg-red-600"
                >
                  Delete List
                </button>
              </div>
              {list.products && list.products.length > 0 ? (
                <ul className="list-disc list-inside text-gray-700 space-y-1 ml-4">
                  {list.products.map((productId) => {
                    const product = products.find(p => p.id === productId);
                    return (
                      <li key={productId} className="flex justify-between items-center">
                        <span>{product ? product.name : `Product (ID: ${productId})`}</span>
                        <button
                          onClick={() => handleRemoveProductFromList(list.id, productId)}
                          className="text-red-500 hover:underline text-xs ml-2"
                        >
                          Remove
                        </button>
                      </li>
                    );
                  })}
                </ul>
              ) : (
                <p className="text-sm text-gray-600">This list is empty. Add products from "Browse Products".</p>
              )}
              <button className="mt-4 px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 shadow-md">
                Add All to Cart (Simulated)
              </button>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

const RFQAndQuotes = () => {
  const { addDocument, getCollectionRef, isFirestoreReady } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [rfqTitle, setRfqTitle] = useState('');
  const [rfqDetails, setRfqDetails] = useState('');
  const [rfqCategory, setRfqCategory] = useState('');
  const [rfqs, setRfqs] = useState([]);
  const [message, setMessage] = useState('');
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (isFirestoreReady && currentUser) {
      const q = query(getCollectionRef('rfqs', false), where('retailerId', '==', currentUser.uid));
      const unsubscribe = onSnapshot(q, (snapshot) => {
        const fetchedRfqs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        // Sort by submittedAt, newest first
        fetchedRfqs.sort((a, b) => (b.submittedAt?.toDate() || 0) - (a.submittedAt?.toDate() || 0));
        setRfqs(fetchedRfqs);
        setLoading(false);
      }, (error) => {
        console.error("Error fetching RFQs:", error);
        setMessage("Error loading RFQs.");
        setLoading(false);
      });
      return () => unsubscribe();
    }
  }, [isFirestoreReady, currentUser, getCollectionRef]);

  const handleSubmitRFQ = async (e) => {
    e.preventDefault();
    if (!isFirestoreReady) {
      setMessage("System not ready. Please try again.");
      return;
    }
    try {
      await addDocument('rfqs', {
        title: rfqTitle,
        details: rfqDetails,
        category: rfqCategory,
        retailerId: currentUser.uid,
        retailerEmail: currentUser.email,
        status: 'Open',
        quotes: [], // Array to store responses from wholesalers
        submittedAt: serverTimestamp(),
      }, false); // RFQs are private to the retailer, but accessible by wholesalers if needed
      setMessage('Your RFQ has been submitted successfully!');
      setRfqTitle('');
      setRfqDetails('');
      setRfqCategory('');
    } catch (error) {
      console.error("Error submitting RFQ:", error);
      setMessage("Failed to submit RFQ.");
    }
  };

  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">RFQ & Quotes</h2>
      <p className="text-gray-700 mb-4">
        Submit requests for quotes (RFQs) for custom or large-volume orders and manage incoming quotes.
      </p>
      
      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      <div className="mb-8 bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">Submit New RFQ</h3>
        <form onSubmit={handleSubmitRFQ} className="space-y-4">
          <input
            type="text"
            placeholder="RFQ Title (e.g., 'Bulk Office Supplies')"
            value={rfqTitle}
            onChange={(e) => setRfqTitle(e.target.value)}
            className="w-full p-2 border rounded-lg"
            required
          />
          <input
            type="text"
            placeholder="Category (e.g., Electronics, Food & Beverage)"
            value={rfqCategory}
            onChange={(e) => setRfqCategory(e.target.value)}
            className="w-full p-2 border rounded-lg"
            required
          />
          <textarea
            placeholder="Detailed requirements (product specs, quantity, delivery timeframe, special requests)"
            value={rfqDetails}
            onChange={(e) => setRfqDetails(e.target.value)}
            rows="6"
            className="w-full p-2 border rounded-lg"
            required
          ></textarea>
          <button
            type="submit"
            className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 shadow-md"
          >
            Submit RFQ
          </button>
        </form>
      </div>

      <div className="bg-yellow-50 p-4 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-yellow-700 mb-3">My RFQ Submissions</h3>
        {rfqs.length === 0 ? (
          <p className="text-center text-gray-600">You haven't submitted any RFQs yet.</p>
        ) : (
          <ul className="space-y-3">
            {rfqs.map((rfq) => (
              <li key={rfq.id} className="bg-white p-3 rounded-lg shadow-sm border border-yellow-200">
                <div className="flex justify-between items-center mb-1">
                  <p className="font-semibold text-gray-800">{rfq.title}</p>
                  <span className={`px-2 py-1 rounded-full text-xs font-medium ${
                    rfq.status === 'Open' ? 'bg-yellow-200 text-yellow-800' :
                    rfq.status === 'Closed' ? 'bg-gray-200 text-gray-800' :
                    'bg-blue-200 text-blue-800'
                  }`}>
                    {rfq.status}
                  </span>
                </div>
                <p className="text-sm text-gray-600">Category: {rfq.category}</p>
                <p className="text-xs text-gray-500">Submitted: {new Date(rfq.submittedAt?.toDate()).toLocaleDateString()}</p>
                <p className="text-sm text-gray-700 mt-2">{rfq.details}</p>
                <div className="mt-3">
                  <h4 className="text-md font-semibold text-gray-700 mb-2">Quotes Received ({rfq.quotes?.length || 0})</h4>
                  {rfq.quotes && rfq.quotes.length > 0 ? (
                    <ul className="list-disc list-inside text-sm text-gray-700 ml-4 space-y-1">
                      {rfq.quotes.map((quote, idx) => (
                        <li key={idx}>
                          **{quote.wholesalerName}**: Price: Ksh {quote.price.toLocaleString()} for {quote.quantity} units. (Lead Time: {quote.leadTime})
                          <button className="ml-2 text-blue-600 hover:underline text-xs">Accept Quote</button>
                        </li>
                      ))}
                    </ul>
                  ) : (
                    <p className="text-xs text-gray-600">No quotes received yet.</p>
                  )}
                </div>
                <button className="px-3 py-1 bg-indigo-600 text-white text-sm rounded-lg hover:bg-indigo-700 mt-3">View Details</button>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};


const IncomingRFQs = () => {
  const { getCollectionRef, isFirestoreReady, setDocument } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [rfqs, setRfqs] = useState([]);
  const [message, setMessage] = useState('');
  const [loading, setLoading] = useState(true);
  const [quoteFormData, setQuoteFormData] = useState({
    price: '',
    quantity: '',
    leadTime: '',
    wholesalerName: currentUser?.email, // Default to current user's email
    wholesalerId: currentUser?.uid,
  });

  useEffect(() => {
    if (isFirestoreReady && currentUser) {
      // To allow wholesalers to see RFQs from *other* users, the RFQ collection
      // needs to be public, or there needs to be a specific rule
      // allowing a wholesaler to read RFQs from other users.
      // For this demo, we'll assume RFQs are stored in a PUBLIC collection
      // so all wholesalers can see them.
      const rfqsColRef = collection(db, `artifacts/${appId}/public/data/rfqs`); // Assuming RFQs are public
      if (!rfqsColRef) {
        setMessage("Error: Could not get RFQs collection reference.");
        setLoading(false);
        return;
      }
      const q = query(rfqsColRef, where('status', '==', 'Open'));
      const unsubscribe = onSnapshot(q, (snapshot) => {
        const fetchedRfqs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        // Filter out RFQs already quoted by this wholesaler for display simplicity
        const filteredForWholesaler = fetchedRfqs.filter(rfq => 
          !rfq.quotes || !rfq.quotes.some(q => q.wholesalerId === currentUser.uid)
        );
        // Sort by submittedAt, newest first
        filteredForWholesaler.sort((a, b) => (b.submittedAt?.toDate() || 0) - (a.submittedAt?.toDate() || 0));
        setRfqs(filteredForWholesaler);
        setLoading(false);
      }, (error) => {
        console.error("Error fetching RFQs:", error);
        setMessage("Error loading RFQs.");
        setLoading(false);
      });
      return () => unsubscribe();
    }
  }, [isFirestoreReady, currentUser, getCollectionRef]);

  const handleQuoteSubmit = async (rfqId, retailerId, existingQuotes = []) => { // Added retailerId
    if (!isFirestoreReady || !quoteFormData.price || !quoteFormData.quantity || !quoteFormData.leadTime) {
      setMessage("Please fill all quote fields.");
      return;
    }

    try {
      // Find the specific RFQ document to update in the retailer's private collection
      const rfqDocRef = doc(db, `artifacts/${appId}/users/${retailerId}/rfqs`, rfqId);
      
      const newQuote = {
        price: parseFloat(quoteFormData.price),
        quantity: parseInt(quoteFormData.quantity),
        leadTime: quoteFormData.leadTime,
        wholesalerId: currentUser.uid,
        wholesalerName: currentUser.email,
        quotedAt: serverTimestamp(),
      };

      await updateDoc(rfqDocRef, {
        quotes: [...(existingQuotes || []), newQuote],
        // Optionally, update status if you want only one quote per RFQ
        // status: 'Quoted' 
      });

      setMessage(`Quote for RFQ ${rfqId} submitted successfully!`);
      setQuoteFormData({ // Clear form data after submission
        price: '',
        quantity: '',
        leadTime: '',
        wholesalerName: currentUser?.email,
        wholesalerId: currentUser?.uid,
      });
      // Optionally remove the quoted RFQ from the list of incoming RFQs
      setRfqs(prevRfqs => prevRfqs.filter(rfq => rfq.id !== rfqId));

    } catch (error) {
      console.error("Error submitting quote:", error);
      setMessage("Failed to submit quote.");
    }
  };

  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">Incoming RFQs</h2>
      <p className="text-gray-700 mb-4">
        Review requests for quotes from retailers and submit your best offers.
      </p>

      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      {rfqs.length === 0 ? (
        <p className="text-center text-gray-600">No open RFQs currently available for you to quote.</p>
      ) : (
        <div className="space-y-6">
          {rfqs.map((rfq) => (
            <div key={rfq.id} className="bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
              <div className="flex justify-between items-center mb-2">
                <h3 className="text-xl font-semibold text-gray-800">{rfq.title}</h3>
                <span className={`px-2 py-1 rounded-full text-xs font-medium bg-yellow-200 text-yellow-800`}>
                  {rfq.status}
                </span>
              </div>
              <p className="text-sm text-gray-600">Category: {rfq.category}</p>
              <p className="text-sm text-gray-600">Requested by: {rfq.retailerEmail}</p>
              <p className="text-xs text-gray-500">Submitted: {new Date(rfq.submittedAt?.toDate()).toLocaleDateString()}</p>
              <p className="text-gray-700 mt-2">{rfq.details}</p>

              <div className="mt-4 p-3 bg-white rounded-lg shadow-inner">
                <h4 className="text-lg font-semibold text-gray-800 mb-3">Submit Your Quote</h4>
                <div className="grid grid-cols-1 md:grid-cols-3 gap-3">
                  <input
                    type="number"
                    placeholder="Your Price"
                    value={quoteFormData.price}
                    onChange={(e) => setQuoteFormData({ ...quoteFormData, price: e.target.value })}
                    className="p-2 border rounded-lg w-full"
                    required
                  />
                  <input
                    type="number"
                    placeholder="Quantity You Can Supply"
                    value={quoteFormData.quantity}
                    onChange={(e) => setQuoteFormData({ ...quoteFormData, quantity: e.target.value })}
                    className="p-2 border rounded-lg w-full"
                    required
                  />
                  <input
                    type="text"
                    placeholder="Lead Time (e.g., 3-5 days)"
                    value={quoteFormData.leadTime}
                    onChange={(e) => setQuoteFormData({ ...quoteFormData, leadTime: e.target.value })}
                    className="p-2 border rounded-lg w-full"
                    required
                  />
                </div>
                <button
                  onClick={() => handleQuoteSubmit(rfq.id, rfq.retailerId, rfq.quotes)} // Pass retailerId
                  className="mt-4 px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 shadow-md"
                >
                  Submit Quote
                </button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};


const WholesalerProducts = () => {
  const { getCollectionRef, addDocument, setDocument, deleteDocument, isFirestoreReady } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [message, setMessage] = useState('');
  const [newProduct, setNewProduct] = useState({
    name: '',
    description: '',
    brand: '',
    category: '',
    price: '',
    moq: '',
    stock: '',
    wholesalerId: currentUser?.uid,
    wholesalerName: currentUser?.email, // Using email as a placeholder name
  });
  const [editingProduct, setEditingProduct] = useState(null);

  useEffect(() => {
    if (isFirestoreReady && currentUser) {
      const q = query(getCollectionRef('products', true), where('wholesalerId', '==', currentUser.uid));
      const unsubscribe = onSnapshot(q, (snapshot) => {
        const fetchedProducts = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        setProducts(fetchedProducts);
        setLoading(false);
      }, (error) => {
        console.error("Error fetching products:", error);
        setMessage("Error loading your products.");
        setLoading(false);
      });
      return () => unsubscribe();
    }
  }, [isFirestoreReady, currentUser, getCollectionRef]);

  const handleAddProduct = async (e) => {
    e.preventDefault();
    if (!isFirestoreReady) {
      setMessage("Please wait, system is initializing.");
      return;
    }
    try {
      await addDocument('products', {
        ...newProduct,
        price: parseFloat(newProduct.price),
        moq: parseInt(newProduct.moq),
        stock: parseInt(newProduct.stock),
      }, true); // Products are public for all to see
      setMessage('Product added successfully!');
      setNewProduct({
        name: '', description: '', brand: '', category: '', price: '', moq: '', stock: '',
        wholesalerId: currentUser?.uid, wholesalerName: currentUser?.email,
      });
    } catch (error) {
      console.error("Error adding product:", error);
      setMessage("Failed to add product.");
    }
  };

  const handleEditProduct = (product) => {
    setEditingProduct({ ...product });
  };

  const handleUpdateProduct = async (e) => {
    e.preventDefault();
    if (!isFirestoreReady || !editingProduct) {
      setMessage("Please wait, system is initializing or no product selected for edit.");
      return;
    }
    try {
      await setDocument('products', editingProduct.id, {
        ...editingProduct,
        price: parseFloat(editingProduct.price),
        moq: parseInt(editingProduct.moq),
        stock: parseInt(editingProduct.stock),
      }, true);
      setMessage('Product updated successfully!');
      setEditingProduct(null);
    } catch (error) {
      console.error("Error updating product:", error);
      setMessage("Failed to update product.");
    }
  };

  const handleDeleteProduct = async (productId) => {
    if (!isFirestoreReady) {
      setMessage("Please wait, system is initializing.");
      return;
    }
    if (window.confirm('Are you sure you want to delete this product?')) {
      try {
        await deleteDocument('products', productId, true);
        setMessage('Product deleted successfully!');
      } catch (error) {
        console.error("Error deleting product:", error);
        setMessage("Failed to delete product.");
      }
    }
  };

  const handleUpdateStock = async (productId, newStock) => {
    if (!isFirestoreReady) {
      setMessage("Please wait, system is initializing.");
      return;
    }
    try {
      await setDocument('products', productId, { stock: parseInt(newStock) }, true);
      setMessage('Stock updated successfully!');
    } catch (error) {
      console.error("Error updating stock:", error);
      setMessage("Failed to update stock.");
    }
  };


  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">My Products</h2>
      <p className="text-gray-700 mb-4">
        Manage your product listings, update inventory, and set pricing.
      </p>

      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      {/* Add/Edit Product Form */}
      <div className="mb-8 bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">
          {editingProduct ? 'Edit Product' : 'Add New Product'}
        </h3>
        <form onSubmit={editingProduct ? handleUpdateProduct : handleAddProduct} className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <input
            type="text"
            placeholder="Product Name"
            value={editingProduct ? editingProduct.name : newProduct.name}
            onChange={(e) => editingProduct ? setEditingProduct({ ...editingProduct, name: e.target.value }) : setNewProduct({ ...newProduct, name: e.target.value })}
            className="p-2 border rounded-lg"
            required
          />
          <input
            type="text"
            placeholder="Brand"
            value={editingProduct ? editingProduct.brand : newProduct.brand}
            onChange={(e) => editingProduct ? setEditingProduct({ ...editingProduct, brand: e.target.value }) : setNewProduct({ ...newProduct, brand: e.target.value })}
            className="p-2 border rounded-lg"
            required
          />
          <input
            type="text"
            placeholder="Category"
            value={editingProduct ? editingProduct.category : newProduct.category}
            onChange={(e) => editingProduct ? setEditingProduct({ ...editingProduct, category: e.target.value }) : setNewProduct({ ...newProduct, category: e.target.value })}
            className="p-2 border rounded-lg"
            required
          />
          <input
            type="number"
            placeholder="Price (Ksh)"
            value={editingProduct ? editingProduct.price : newProduct.price}
            onChange={(e) => editingProduct ? setEditingProduct({ ...editingProduct, price: e.target.value }) : setNewProduct({ ...newProduct, price: e.target.value })}
            className="p-2 border rounded-lg"
            required
          />
          <input
            type="number"
            placeholder="Min. Order Quantity (MOQ)"
            value={editingProduct ? editingProduct.moq : newProduct.moq}
            onChange={(e) => editingProduct ? setEditingProduct({ ...editingProduct, moq: e.target.value }) : setNewProduct({ ...newProduct, moq: e.target.value })}
            className="p-2 border rounded-lg"
            required
          />
          <input
            type="number"
            placeholder="Stock Quantity"
            value={editingProduct ? editingProduct.stock : newProduct.stock}
            onChange={(e) => editingProduct ? setEditingProduct({ ...editingProduct, stock: e.target.value }) : setNewProduct({ ...newProduct, stock: e.target.value })}
            className="p-2 border rounded-lg"
            required
          />
          <textarea
            placeholder="Description"
            value={editingProduct ? editingProduct.description : newProduct.description}
            onChange={(e) => editingProduct ? setEditingProduct({ ...editingProduct, description: e.target.value }) : setNewProduct({ ...newProduct, description: e.target.value })}
            rows="3"
            className="p-2 border rounded-lg col-span-full"
            required
          ></textarea>
          <div className="col-span-full flex justify-end space-x-2">
            {editingProduct && (
              <button
                type="button"
                onClick={() => setEditingProduct(null)}
                className="px-4 py-2 bg-gray-500 text-white rounded-lg hover:bg-gray-600 shadow-md"
              >
                Cancel Edit
              </button>
            )}
            <button
              type="submit"
              className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 shadow-md"
            >
              {editingProduct ? 'Update Product' : 'Add Product'}
            </button>
          </div>
        </form>
      </div>

      {/* Product List */}
      <h3 className="text-xl font-semibold text-gray-800 mb-3">Your Current Listings</h3>
      {products.length === 0 ? (
        <p className="text-center text-gray-600">You have no products listed yet.</p>
      ) : (
        <div className="overflow-x-auto">
          <table className="min-w-full bg-white rounded-lg overflow-hidden shadow-sm">
            <thead className="bg-gray-200">
              <tr>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Name</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Brand</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Category</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Price</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">MOQ</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Stock</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Actions</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Update Stock</th>
              </tr>
            </thead>
            <tbody>
              {products.map((product) => (
                <tr key={product.id} className="border-b border-gray-200 hover:bg-gray-100">
                  <td className="py-2 px-4 text-sm text-gray-700">{product.name}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{product.brand}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{product.category}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">Ksh {product.price.toLocaleString()}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{product.moq}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{product.stock}</td>
                  <td className="py-2 px-4 text-sm">
                    <button onClick={() => handleEditProduct(product)} className="text-blue-600 hover:underline mr-2">Edit</button>
                    <button onClick={() => handleDeleteProduct(product.id)} className="text-red-600 hover:underline">Delete</button>
                  </td>
                  <td className="py-2 px-4 text-sm">
                    <input
                      type="number"
                      defaultValue={product.stock}
                      onBlur={(e) => handleUpdateStock(product.id, e.target.value)}
                      className="w-24 p-1 border rounded-md text-center text-sm"
                    />
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
};

const WholesalerAnalytics = () => (
  <div className="p-6 bg-white rounded-xl shadow-md">
    <h2 className="text-3xl font-bold text-gray-800 mb-6">Performance Analytics</h2>
    <p className="text-gray-700 mb-4">
      Gain insights into your sales performance, popular products, and customer behavior on GOOB HUB.
    </p>
    <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
      <div className="bg-green-50 p-6 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-green-700 mb-2">Your Sales Overview (Simulated)</h3>
        <div className="h-48 bg-green-100 flex items-center justify-center text-gray-600 rounded-lg">
          [Placeholder for Your Sales Chart]
        </div>
        <p className="text-sm text-gray-600 mt-2">Total revenue, units sold, average order value for your products.</p>
      </div>
      <div className="bg-purple-50 p-6 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-purple-700 mb-2">Top Selling Products (Your Listings)</h3>
        <ul className="list-disc list-inside text-gray-700 space-y-1">
          <li>Your Product X - 500 units sold</li>
          <li>Your Product Y - 300 units sold</li>
        </ul>
        <p className="text-sm text-gray-600 mt-2">Identify your top performers and optimize inventory.</p>
      </div>
      <div className="bg-orange-50 p-6 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-orange-700 mb-2">Customer Insights (Simulated)</h3>
        <ul className="list-disc list-inside text-gray-700 space-y-1">
          <li>Top Retailers: Retailer A, Retailer B</li>
          <li>Average Purchase Frequency: 2 times/month</li>
        </ul>
        <p className="text-sm text-gray-600 mt-2">Understand your buyers' behavior and loyalty.</p>
      </div>
      <div className="bg-pink-50 p-6 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-pink-700 mb-2">Inventory Forecast (AI Integration Placeholder)</h3>
        <p className="text-gray-700">
          Our AI can predict future demand for your products, helping you optimize stock levels
          and avoid shortages or overstocking.
        </p>
        <button className="mt-3 px-4 py-2 bg-pink-600 text-white rounded-lg hover:bg-pink-700">
          Generate Forecast
        </button>
      </div>
    </div>
  </div>
);

const ManageInventory = () => (
  <div className="p-6 bg-white rounded-xl shadow-md">
    <h2 className="text-3xl font-bold text-gray-800 mb-6">Manage Inventory</h2>
    <p className="text-gray-700 mb-4">
      Keep your stock levels accurate and up-to-date across all your products.
      This helps prevent overselling and ensures smooth order fulfillment.
    </p>
    <div className="bg-blue-50 p-4 rounded-lg shadow-sm">
      <h3 className="text-xl font-semibold text-blue-700 mb-3">Current Stock Levels (Simulated)</h3>
      <table className="min-w-full bg-white rounded-lg overflow-hidden">
        <thead className="bg-gray-200">
          <tr>
            <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Product Name</th>
            <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Current Stock</th>
            <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Min Stock Alert</th>
            <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Actions</th>
          </tr>
        </thead>
        <tbody>
          <tr className="border-b border-gray-200 hover:bg-gray-100">
            <td className="py-2 px-4 text-sm text-gray-700">Product A</td>
            <td className="py-2 px-4 text-sm text-green-600">1500</td>
            <td className="py-2 px-4 text-sm text-gray-700">200</td>
            <td className="py-2 px-4 text-sm">
              <button className="text-blue-600 hover:underline mr-2">Update Stock</button>
            </td>
          </tr>
          <tr className="border-b border-gray-200 hover:bg-gray-100">
            <td className="py-2 px-4 text-sm text-gray-700">Product B</td>
            <td className="py-2 px-4 text-sm text-orange-600">80</td>
            <td className="py-2 px-4 text-sm text-gray-700">100</td>
            <td className="py-2 px-4 text-sm">
              <button className="text-blue-600 hover:underline mr-2">Update Stock</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    <div className="mt-6 p-4 bg-gray-50 rounded-lg border border-gray-200">
      <h4 className="text-lg font-semibold text-gray-800 mb-2">Real-time Inventory Sync (Integration Placeholder)</h4>
      <p className="text-gray-700">
        Integrate your existing ERP or WMS to automatically synchronize stock levels
        and avoid manual updates.
      </p>
      <button className="mt-3 px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700">
        Connect ERP System
      </button>
    </div>
  </div>
);

const WholesalerOffers = () => {
  const { isFirestoreReady, getCollectionRef, addDocument, setDocument, deleteDocument } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [offers, setOffers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [message, setMessage] = useState('');
  const [newOffer, setNewOffer] = useState({
    name: '',
    type: 'Product Specific', // 'Product Specific', 'Category Specific', 'Customer Group Specific'
    target: '', // Product ID, Category Name, Customer Group Name
    discountPercentage: '',
    startDate: '',
    endDate: '',
    wholesalerId: currentUser?.uid,
    wholesalerName: currentUser?.email,
  });
  const [editingOffer, setEditingOffer] = useState(null);

  useEffect(() => {
    if (isFirestoreReady && currentUser) {
      const offersColRef = getCollectionRef('offers', false); // Offers are private to wholesaler
      if (offersColRef) {
        const q = query(offersColRef, where('wholesalerId', '==', currentUser.uid));
        const unsubscribe = onSnapshot(q, (snapshot) => {
          setOffers(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
          setLoading(false);
        }, (error) => {
          console.error("Error fetching offers:", error);
          setMessage("Error loading offers.");
          setLoading(false);
        });
        return () => unsubscribe();
      } else {
        setLoading(false);
      }
    }
  }, [isFirestoreReady, currentUser, getCollectionRef]);

  const handleAddOffer = async (e) => {
    e.preventDefault();
    if (!isFirestoreReady || !newOffer.name || !newOffer.discountPercentage || !newOffer.startDate || !newOffer.endDate) {
      setMessage("Please fill all required offer fields.");
      return;
    }
    try {
      await addDocument('offers', {
        ...newOffer,
        discountPercentage: parseFloat(newOffer.discountPercentage),
        startDate: new Date(newOffer.startDate).toISOString(),
        endDate: new Date(newOffer.endDate).toISOString(),
      }, false);
      setMessage('Offer added successfully!');
      setNewOffer({
        name: '', type: 'Product Specific', target: '', discountPercentage: '', startDate: '', endDate: '',
        wholesalerId: currentUser?.uid, wholesalerName: currentUser?.email,
      });
    } catch (error) {
      console.error("Error adding offer:", error);
      setMessage("Failed to add offer.");
    }
  };

  const handleEditOffer = (offer) => {
    setEditingOffer({ ...offer, startDate: offer.startDate?.split('T')[0], endDate: offer.endDate?.split('T')[0] }); // Format dates for input
  };

  const handleUpdateOffer = async (e) => {
    e.preventDefault();
    if (!isFirestoreReady || !editingOffer) return;
    try {
      await setDocument('offers', editingOffer.id, {
        ...editingOffer,
        discountPercentage: parseFloat(editingOffer.discountPercentage),
        startDate: new Date(editingOffer.startDate).toISOString(),
        endDate: new Date(editingOffer.endDate).toISOString(),
      }, false);
      setMessage('Offer updated successfully!');
      setEditingOffer(null);
    } catch (error) {
      console.error("Error updating offer:", error);
      setMessage("Failed to update offer.");
    }
  };

  const handleDeleteOffer = async (offerId) => {
    if (!isFirestoreReady) return;
    if (window.confirm('Are you sure you want to delete this offer?')) {
      try {
        await deleteDocument('offers', offerId, false);
        setMessage('Offer deleted successfully!');
      } catch (error) {
        console.error("Error deleting offer:", error);
        setMessage("Failed to delete offer.");
      }
    }
  };

  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">Manage Offers & Pricing</h2>
      <p className="text-gray-700 mb-4">
        Create custom pricing tiers, volume discounts, and special offers for different buyer groups.
      </p>

      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      {/* Add/Edit Offer Form */}
      <div className="mb-8 bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">
          {editingOffer ? 'Edit Offer' : 'Create New Offer'}
        </h3>
        <form onSubmit={editingOffer ? handleUpdateOffer : handleAddOffer} className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <input
            type="text"
            placeholder="Offer Name (e.g., 'Seasonal Discount')"
            value={editingOffer ? editingOffer.name : newOffer.name}
            onChange={(e) => editingOffer ? setEditingOffer({ ...editingOffer, name: e.target.value }) : setNewOffer({ ...newOffer, name: e.target.value })}
            className="p-2 border rounded-lg"
            required
          />
          <select
            value={editingOffer ? editingOffer.type : newOffer.type}
            onChange={(e) => editingOffer ? setEditingOffer({ ...editingOffer, type: e.target.value }) : setNewOffer({ ...newOffer, type: e.target.value })}
            className="p-2 border rounded-lg"
          >
            <option>Product Specific</option>
            <option>Category Specific</option>
            <option>Customer Group Specific</option>
          </select>
          <input
            type="text"
            placeholder="Target (e.g., Product ID, Category Name, Group Name)"
            value={editingOffer ? editingOffer.target : newOffer.target}
            onChange={(e) => editingOffer ? setEditingOffer({ ...editingOffer, target: e.target.value }) : setNewOffer({ ...newOffer, target: e.target.value })}
            className="p-2 border rounded-lg"
          />
          <input
            type="number"
            placeholder="Discount Percentage (e.g., 10)"
            value={editingOffer ? editingOffer.discountPercentage : newOffer.discountPercentage}
            onChange={(e) => editingOffer ? setEditingOffer({ ...editingOffer, discountPercentage: e.target.value }) : setNewOffer({ ...newOffer, discountPercentage: e.target.value })}
            className="p-2 border rounded-lg"
            required
          />
          <div>
            <label className="block text-sm text-gray-700">Start Date</label>
            <input
              type="date"
              value={editingOffer ? editingOffer.startDate : newOffer.startDate}
              onChange={(e) => editingOffer ? setEditingOffer({ ...editingOffer, startDate: e.target.value }) : setNewOffer({ ...newOffer, startDate: e.target.value })}
              className="w-full p-2 border rounded-lg"
              required
            />
          </div>
          <div>
            <label className="block text-sm text-gray-700">End Date</label>
            <input
              type="date"
              value={editingOffer ? editingOffer.endDate : newOffer.endDate}
              onChange={(e) => editingOffer ? setEditingOffer({ ...editingOffer, endDate: e.target.value }) : setNewOffer({ ...newOffer, endDate: e.target.value })}
              className="w-full p-2 border rounded-lg"
              required
            />
          </div>
          <div className="col-span-full flex justify-end space-x-2">
            {editingOffer && (
              <button
                type="button"
                onClick={() => setEditingOffer(null)}
                className="px-4 py-2 bg-gray-500 text-white rounded-lg hover:bg-gray-600 shadow-md"
              >
                Cancel Edit
              </button>
            )}
            <button
              type="submit"
              className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 shadow-md"
            >
              {editingOffer ? 'Update Offer' : 'Create Offer'}
            </button>
          </div>
        </form>
      </div>

      {/* Offers List */}
      <h3 className="text-xl font-semibold text-gray-800 mb-3">Your Active Offers</h3>
      {offers.length === 0 ? (
        <p className="text-center text-gray-600">You have no offers created yet.</p>
      ) : (
        <div className="overflow-x-auto">
          <table className="min-w-full bg-white rounded-lg overflow-hidden shadow-sm">
            <thead className="bg-gray-200">
              <tr>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Name</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Type</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Target</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Discount (%)</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Period</th>
                <th className="py-2 px-4 text-left text-sm font-semibold text-gray-700">Actions</th>
              </tr>
            </thead>
            <tbody>
              {offers.map((offer) => (
                <tr key={offer.id} className="border-b border-gray-200 hover:bg-gray-100">
                  <td className="py-2 px-4 text-sm text-gray-700">{offer.name}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{offer.type}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{offer.target || 'N/A'}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">{offer.discountPercentage}</td>
                  <td className="py-2 px-4 text-sm text-gray-700">
                    {new Date(offer.startDate).toLocaleDateString()} - {new Date(offer.endDate).toLocaleDateString()}
                  </td>
                  <td className="py-2 px-4 text-sm">
                    <button onClick={() => handleEditOffer(offer)} className="text-blue-600 hover:underline mr-2">Edit</button>
                    <button onClick={() => handleDeleteOffer(offer.id)} className="text-red-600 hover:underline">Delete</button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
      <div className="mt-6 p-4 bg-gray-50 rounded-lg border border-gray-200">
        <h4 className="text-lg font-semibold text-gray-800 mb-2">Dynamic Pricing (AI Integration Placeholder)</h4>
        <p className="text-gray-700">
          Our AI can analyze market conditions and competitor pricing to suggest optimal dynamic pricing strategies
          for your products to maximize sales and profit margins.
        </p>
        <button className="mt-3 px-4 py-2 bg-purple-600 text-white rounded-lg hover:bg-purple-700">
          Activate Dynamic Pricing
        </button>
      </div>
    </div>
  );
};

const DistributorOrders = () => (
  <div className="p-6 bg-white rounded-xl shadow-md">
    <h2 className="text-3xl font-bold text-gray-800 mb-6">Distributor Orders (from Wholesalers)</h2>
    <p className="text-gray-700 mb-4">
      As a distributor, you can also place bulk orders directly from wholesalers on the platform.
      This section tracks your purchases from other suppliers.
    </p>
    <div className="bg-blue-50 p-4 rounded-lg shadow-sm">
      <h3 className="text-xl font-semibold text-blue-700 mb-3">My Purchases from Wholesalers (Simulated)</h3>
      <ul className="space-y-3">
        <li className="flex justify-between items-center bg-white p-3 rounded-lg shadow-sm border border-blue-200">
          <div>
            <p className="font-semibold text-gray-800">Order #D-001: Raw Materials Batch</p>
            <p className="text-sm text-gray-600">Supplier: "MegaWholesale Inc."</p>
            <p className="text-xs text-green-500">Status: Delivered</p>
          </div>
          <button className="px-3 py-1 bg-blue-600 text-white text-sm rounded-lg hover:bg-blue-700">View Details</button>
        </li>
        <li className="flex justify-between items-center bg-white p-3 rounded-lg shadow-sm border border-yellow-200">
          <div>
            <p className="font-semibold text-gray-800">Order #D-002: Component Parts</p>
            <p className="text-sm text-gray-600">Supplier: "GlobalParts Co."</p>
            <p className="text-xs text-yellow-600">Status: Shipped</p>
          </div>
          <button className="px-3 py-1 bg-yellow-600 text-white text-sm rounded-lg hover:bg-yellow-700">View Details</button>
        </li>
      </ul>
    </div>
    <div className="mt-6">
      <h3 className="text-xl font-semibold text-gray-800 mb-3">Place New Order from Wholesaler</h3>
      <p className="text-gray-700 mb-3">
        You can browse products in the "Browse Products" section and place orders.
        This section specifically tracks your purchases *as a distributor* from other wholesalers.
      </p>
      <button className="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700" onClick={() => console.log('Navigate to browse products for distributors')}>
        Browse Wholesaler Products
      </button>
    </div>
  </div>
);

// New Component: Notifications
const Notifications = () => {
  const { isFirestoreReady, getCollectionRef } = useContext(FirestoreContext);
  const { currentUser } = useContext(AuthContext);
  const [notifications, setNotifications] = useState([]);
  const [loading, setLoading] = useState(true);
  const [message, setMessage] = useState('');

  useEffect(() => {
    if (isFirestoreReady && currentUser) {
      // For demo, we'll fetch all messages as notifications.
      // In a real system, you'd have a dedicated 'notifications' collection
      // for each user, or a more sophisticated system.
      const q = query(
        getCollectionRef('messages', true), // Assuming messages are public and can serve as notifications
        where('receiverId', '==', currentUser.uid)
        // orderBy('timestamp', 'desc') // Requires index
      );
      const unsubscribe = onSnapshot(q, (snapshot) => {
        const fetchedNotifications = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data(), type: 'message' }));
        // Add some simulated notifications
        const simulatedNotifications = [
          { id: 'sim1', type: 'order_update', content: 'Order #GH-2025-00123 has been shipped!', timestamp: new Date(Date.now() - 3600000) },
          { id: 'sim2', type: 'rfq_response', content: 'You received a new quote for RFQ #RFQ-001.', timestamp: new Date(Date.now() - 7200000) },
        ].filter(sim => !fetchedNotifications.some(notif => notif.id === sim.id)); // Avoid duplicates if IDs clash
        
        const combinedNotifications = [...fetchedNotifications, ...simulatedNotifications];
        combinedNotifications.sort((a, b) => (b.timestamp?.toDate ? b.timestamp.toDate() : b.timestamp) - (a.timestamp?.toDate ? a.timestamp.toDate() : a.timestamp));
        setNotifications(combinedNotifications);
        setLoading(false);
      }, (error) => {
        console.error("Error fetching notifications:", error);
        setMessage("Error loading notifications.");
        setLoading(false);
      });
      return () => unsubscribe();
    }
  }, [isFirestoreReady, currentUser, getCollectionRef]);

  if (loading) return <LoadingSpinner />;

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">Notifications</h2>
      <p className="text-gray-700 mb-4">
        Stay updated with important activities on your GOOB HUB account.
      </p>

      {message && (
        <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
          {message}
        </div>
      )}

      {notifications.length === 0 ? (
        <p className="text-center text-gray-600">No new notifications.</p>
      ) : (
        <div className="space-y-4">
          {notifications.map((notification) => (
            <div key={notification.id} className="bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
              <div className="flex items-center">
                <span className={`text-xl mr-3 ${
                  notification.type === 'message' ? 'text-blue-500' :
                  notification.type === 'order_update' ? 'text-green-500' :
                  notification.type === 'rfq_response' ? 'text-yellow-500' :
                  'text-gray-500'
                }`}>
                  {notification.type === 'message' && '✉️'}
                  {notification.type === 'order_update' && '🚚'}
                  {notification.type === 'rfq_response' && '💬'}
                </span>
                <div>
                  <p className="font-semibold text-gray-800">
                    {notification.type === 'message' && `New Message from ${notification.senderEmail?.split('@')[0]}: ${notification.content}`}
                    {notification.type === 'order_update' && notification.content}
                    {notification.type === 'rfq_response' && notification.content}
                  </p>
                  <p className="text-xs text-gray-500 mt-1">
                    {notification.timestamp ? new Date(notification.timestamp.toDate ? notification.timestamp.toDate() : notification.timestamp).toLocaleString() : 'N/A'}
                  </p>
                </div>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};


const MyAccount = () => {
  const { currentUser } = useContext(AuthContext);
  const { userId } = useContext(AuthContext);
  const [profileData, setProfileData] = useState({
    companyName: '',
    contactPerson: '',
    phone: '',
    address: '',
    role: '',
  });
  const [message, setMessage] = useState('');
  const { setDocument, getDocRef, isFirestoreReady } = useContext(FirestoreContext);

  useEffect(() => {
    const fetchProfile = async () => {
      if (isFirestoreReady && currentUser) {
        try {
          const docSnap = await getDoc(getDocRef('userProfiles', currentUser.uid, false));
          if (docSnap.exists()) {
            setProfileData(docSnap.data());
          } else {
            // Initialize profile data if it doesn't exist
            setProfileData({
              companyName: '',
              contactPerson: currentUser.email, // Default contact person to email
              phone: '',
              address: '',
              role: '', // User will select
            });
            setMessage("No profile data found. Please update your profile.");
          }
        } catch (error) {
          console.error("Error fetching profile:", error);
          setMessage("Error loading profile data.");
        }
      }
    };
    fetchProfile();
  }, [isFirestoreReady, currentUser, getDocRef]);

  const handleProfileUpdate = async (e) => {
    e.preventDefault();
    if (!isFirestoreReady || !currentUser) {
      setMessage("System not ready or user not logged in.");
      return;
    }
    try {
      await setDocument('userProfiles', currentUser.uid, profileData, false);
      setMessage("Profile updated successfully!");
    } catch (error) {
      console.error("Error updating profile:", error);
      setMessage("Failed to update profile.");
    }
  };

  const handleChange = (e) => {
    const { name, value } = e.target;
    setProfileData(prev => ({ ...prev, [name]: value }));
  };

  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-3xl font-bold text-gray-800 mb-6">My Account</h2>
      <p className="text-gray-700 mb-4">
        Manage your personal and company information, and view your account settings.
      </p>
      <div className="bg-gray-50 p-4 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">Account Details</h3>
        <p className="text-gray-700">Email: <span className="font-medium">{currentUser?.email}</span></p>
        <p className="text-gray-700">User ID: <span className="font-medium">{userId}</span></p>
        <p className="text-sm text-gray-600 mt-2">
          Your unique User ID is displayed for reference.
        </p>
      </div>

      <div className="mt-6 bg-gray-50 p-4 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">Company Profile</h3>
        {message && (
          <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-lg mb-4" role="alert">
            {message}
          </div>
        )}
        <form onSubmit={handleProfileUpdate} className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div>
            <label htmlFor="companyName" className="block text-sm font-medium text-gray-700">Company Name</label>
            <input
              type="text"
              id="companyName"
              name="companyName"
              value={profileData.companyName}
              onChange={handleChange}
              className="w-full p-2 border rounded-lg"
              required
            />
          </div>
          <div>
            <label htmlFor="contactPerson" className="block text-sm font-medium text-gray-700">Contact Person</label>
            <input
              type="text"
              id="contactPerson"
              name="contactPerson"
              value={profileData.contactPerson}
              onChange={handleChange}
              className="w-full p-2 border rounded-lg"
              required
            />
          </div>
          <div>
            <label htmlFor="phone" className="block text-sm font-medium text-gray-700">Phone Number</label>
            <input
              type="tel"
              id="phone"
              name="phone"
              value={profileData.phone}
              onChange={handleChange}
              className="w-full p-2 border rounded-lg"
              required
            />
          </div>
          <div>
            <label htmlFor="address" className="block text-sm font-medium text-gray-700">Address</label>
            <input
              type="text"
              id="address"
              name="address"
              value={profileData.address}
              onChange={handleChange}
              className="w-full p-2 border rounded-lg"
              required
            />
          </div>
          <div className="md:col-span-2">
            <label htmlFor="role" className="block text-sm font-medium text-gray-700">Your Role (Self-Declared)</label>
            <select
              id="role"
              name="role"
              value={profileData.role}
              onChange={handleChange}
              className="w-full p-2 border rounded-lg"
              required
            >
              <option value="">Select Role</option>
              <option value="retailer">Retailer</option>
              <option value="distributor">Distributor</option>
              <option value="wholesaler">Wholesaler</option>
            </select>
          </div>
          <div className="md:col-span-2 flex justify-end">
            <button
              type="submit"
              className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 shadow-md"
            >
              Update Profile
            </button>
          </div>
        </form>
      </div>
    </div>
  );
};


// --- Main App Component ---
const App = () => {
  const { currentUser, userId, loadingAuth, logout } = useContext(AuthContext);
  const { isFirestoreReady, getCollectionRef } = useContext(FirestoreContext); // Destructure getCollectionRef
  const [currentView, setCurrentView] = useState('admin-dashboard'); // Default view
  const [userRole, setUserRole] = useState('retailer'); // Default role for demonstration

  // Simulate user role based on email or some other logic
  useEffect(() => {
    if (currentUser && currentUser.email) {
      if (currentUser.email.includes('admin')) {
        setUserRole('admin');
        setCurrentView('admin-dashboard');
      } else if (currentUser.email.includes('wholesaler')) {
        setUserRole('wholesaler');
        setCurrentView('wholesaler-products');
      } else if (currentUser.email.includes('distributor')) {
        // Distributors might have different initial views or functions
        setUserRole('wholesaler'); // For now, map distributor to wholesaler portal for product/order management
        setCurrentView('wholesaler-products');
      } else {
        setUserRole('retailer');
        setCurrentView('retailer-products');
      }
    } else if (!loadingAuth && !currentUser) {
      // If no authenticated user (e.g., anonymous sign-in), default to retailer view
      setUserRole('retailer');
      setCurrentView('retailer-products');
    }
  }, [currentUser, loadingAuth]);

  // Function to create initial mock products if none exist
  useEffect(() => {
    const createInitialProducts = async () => {
      if (isFirestoreReady && currentUser && currentUser.email?.includes('wholesaler')) {
        // Ensure getCollectionRef is available before calling it
        if (!getCollectionRef) {
          console.error("getCollectionRef is not available yet.");
          return;
        }
        const productsRef = getCollectionRef('products', true);
        const snapshot = await getDocs(productsRef);
        if (snapshot.empty) {
          console.log("No products found for wholesaler. Adding mock products...");
          const mockProducts = [
            {
              name: "Bulk Basmati Rice (25kg)",
              description: "Premium quality Basmati rice, ideal for restaurants and large families.",
              brand: "Golden Grain",
              category: "Food & Beverage",
              price: 3500,
              moq: 10,
              stock: 500,
              wholesalerId: currentUser.uid,
              wholesalerName: currentUser.email,
            },
            {
              name: "Cooking Oil (20L Jerrycan)",
              description: "Pure vegetable cooking oil, suitable for deep frying and general cooking.",
              brand: "Sunny Harvest",
              category: "Food & Beverage",
              price: 2800,
              moq: 5,
              stock: 300,
              wholesalerId: currentUser.uid,
              wholesalerName: currentUser.email,
            },
            {
              name: "Assorted Soft Drinks (24-pack)",
              description: "Mixed pack of popular carbonated soft drinks.",
              brand: "FizzPop Co.",
              category: "Food & Beverage",
              price: 1200,
              moq: 20,
              stock: 1000,
              wholesalerId: currentUser.uid,
              wholesalerName: currentUser.email,
            },
            {
              name: "LED Light Bulbs (Pack of 10)",
              description: "Energy-efficient LED bulbs, 9W, E27 base.",
              brand: "BrightLite",
              category: "Electronics",
              price: 850,
              moq: 50,
              stock: 1200,
              wholesalerId: currentUser.uid,
              wholesalerName: currentUser.email,
            },
            {
              name: "Office Staplers (Box of 12)",
              description: "Heavy-duty staplers with ergonomic design.",
              brand: "OfficeMax",
              category: "Office Supplies",
              price: 1500,
              moq: 10,
              stock: 200,
              wholesalerId: currentUser.uid,
              wholesalerName: currentUser.email,
            },
          ];

          for (const product of mockProducts) {
            await addDoc(productsRef, product);
          }
          console.log("Mock products added.");
        }
      }
    };
    // Add getCollectionRef to the dependency array of this useEffect
    createInitialProducts();
  }, [isFirestoreReady, currentUser, getCollectionRef]);


  const renderContent = () => {
    switch (currentView) {
      // Admin Portal
      case 'admin-dashboard': return <AdminDashboard />;
      case 'admin-users': return <ManageUsers />;
      case 'admin-analytics': return <PlatformAnalytics />;
      case 'admin-disputes': return <DisputeResolution />;
      case 'admin-products-overview': return <ProductOverview />; // New Admin view
      case 'admin-settings': return (
        <div className="p-6 bg-white rounded-xl shadow-md">
          <h2 className="text-3xl font-bold text-gray-800 mb-6">Admin Settings</h2>
          <p className="text-gray-700">Manage platform-wide configurations.</p>
        </div>
      );

      // Retailer Portal
      case 'retailer-products': return <RetailerProducts />;
      case 'retailer-orders': return <MyOrders role="retailer" />;
      case 'retailer-saved-lists': return <SavedLists />;
      case 'retailer-rfq': return <RFQAndQuotes />;
      case 'retailer-account': return <MyAccount />;

      // Wholesaler/Distributor Portal
      case 'wholesaler-products': return <WholesalerProducts />;
      case 'wholesaler-orders': return <MyOrders role="wholesaler" />;
      case 'wholesaler-analytics': return <WholesalerAnalytics />;
      case 'wholesaler-inventory': return <ManageInventory />;
      case 'wholesaler-offers': return <WholesalerOffers />;
      case 'wholesaler-rfqs': return <IncomingRFQs />;
      case 'wholesaler-distributor-orders': return <DistributorOrders />;

      // Global Components
      case 'user-notifications': return <Notifications />;

      default: return <AdminDashboard />;
    }
  };

  if (loadingAuth) {
    return <LoadingSpinner />;
  }

  // If user is not logged in with email/password, show AuthForm
  if (!currentUser || currentUser.isAnonymous) {
    return <AuthForm onLoginSuccess={() => { /* Optionally navigate after login */ }} />;
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 font-sans text-gray-900">
      <Header userEmail={currentUser?.email} onLogout={logout} userId={userId} setView={setCurrentView} />
      <div className="flex flex-col lg:flex-row p-4 gap-4">
        <Sidebar currentView={currentView} setView={setCurrentView} userRole={userRole} />
        <main className="flex-1">
          {renderContent()}
        </main>
      </div>
    </div>
  );
};

// Wrap App with Providers
const WrappedApp = () => (
  <AuthProvider>
    <FirestoreProvider>
      <App />
    </FirestoreProvider>
  </AuthProvider>
);

export default WrappedApp;
