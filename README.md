<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>1-on-1 Chat App</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link
    rel="stylesheet"
    href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.3/css/all.min.css"
  />
  <script defer src="https://unpkg.com/react@18/umd/react.development.js"></script>
  <script defer src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
  <script defer src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <style>
    /* Scrollbar for chat messages */
    .scrollbar-thin::-webkit-scrollbar {
      width: 6px;
      height: 6px;
    }
    .scrollbar-thin::-webkit-scrollbar-thumb {
      background-color: rgba(100, 100, 100, 0.5);
      border-radius: 3px;
    }
  </style>
</head>
<body class="bg-gray-50 dark:bg-gray-900 text-gray-900 dark:text-gray-100 transition-colors duration-300">
  <div id="root" class="h-screen flex flex-col"></div>

  <script type="text/babel">
    const { useState, useEffect, useRef } = React;

    // Helper: Save and load from localStorage for auth token and user info
    const authStorageKey = "chatapp_auth";

    // WebSocket URL (adjust if backend hosted elsewhere)
    const WS_URL = (location.protocol === "https:" ? "wss://" : "ws://") + location.host + "/ws";

    // API base URL (same origin assumed)
    const API_BASE = "";

    // Format timestamp to readable string
    function formatTimestamp(ts) {
      const d = new Date(ts);
      return d.toLocaleString(undefined, {
        hour: "2-digit",
        minute: "2-digit",
        day: "2-digit",
        month: "short",
        year: "numeric",
      });
    }

    // Login Component
    function Login({ onLogin }) {
      const [username, setUsername] = useState("");
      const [password, setPassword] = useState("");
      const [error, setError] = useState(null);
      const [loading, setLoading] = useState(false);

      async function handleSubmit(e) {
        e.preventDefault();
        setError(null);
        if (!username.trim() || !password.trim()) {
          setError("Username and password are required.");
          return;
        }
        setLoading(true);
        try {
          const res = await fetch(API_BASE + "/api/auth/login", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ username, password }),
          });
          if (!res.ok) {
            const data = await res.json();
            throw new Error(data.message || "Login failed");
          }
          const data = await res.json();
          onLogin(data.token, data.user);
        } catch (err) {
          setError(err.message);
        } finally {
          setLoading(false);
        }
      }

      return (
        <div className="flex flex-col justify-center items-center h-full px-4">
          <div className="w-full max-w-md bg-white dark:bg-gray-800 rounded-lg shadow-md p-8">
            <h1 className="text-3xl font-semibold mb-6 text-center">Login to Chat</h1>
            {error && (
              <div className="mb-4 text-red-600 dark:text-red-400 font-medium">{error}</div>
            )}
            <form onSubmit={handleSubmit} className="space-y-4">
              <div>
                <label htmlFor="username" className="block mb-1 font-medium">
                  Masukkan Nama
                </label>
                <input
                  id="username"
                  type="text"
                  className="w-full px-3 py-2 rounded border border-gray-300 dark:border-gray-600 bg-gray-50 dark:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
                  value={username}
                  onChange={(e) => setUsername(e.target.value)}
                  autoComplete="username"
                  required
                />
              </div>
              <div>
                <label htmlFor="password" className="block mb-1 font-medium">
                  Password
                </label>
                <input
                  id="password"
                  type="password"
                  className="w-full px-3 py-2 rounded border border-gray-300 dark:border-gray-600 bg-gray-50 dark:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
                  value={password}
                  onChange={(e) => setPassword(e.target.value)}
                  autoComplete="current-password"
                  required
                />
              </div>
              <button
                type="submit"
                disabled={loading}
                className="w-full bg-blue-600 hover:bg-blue-700 disabled:bg-blue-400 text-white font-semibold py-2 rounded transition"
              >
                {loading ? "Logging in..." : "Login"}
              </button>
            </form>
          </div>
        </div>
      );
    }

    // Chat message bubble component
    function MessageBubble({ message, isOwn }) {
      return (
        <div
          className={`flex ${isOwn ? "justify-end" : "justify-start"} mb-2`}
          aria-label={isOwn ? "Sent message" : "Received message"}
        >
          <div
            className={`max-w-xs md:max-w-md px-4 py-2 rounded-lg break-words whitespace-pre-wrap ${
              isOwn
                ? "bg-blue-600 text-white rounded-br-none"
                : "bg-gray-300 dark:bg-gray-700 text-gray-900 dark:text-gray-100 rounded-bl-none"
            }`}
          >
            <div>{message.text}</div>
            <div className="text-xs text-gray-200 dark:text-gray-400 mt-1 text-right select-none">
              {formatTimestamp(message.createdAt)}
            </div>
          </div>
        </div>
      );
    }

    // User list item component
    function UserListItem({ user, isSelected, onClick }) {
      return (
        <button
          onClick={() => onClick(user)}
          className={`flex items-center space-x-3 px-4 py-3 w-full text-left rounded hover:bg-gray-200 dark:hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500 ${
            isSelected ? "bg-blue-100 dark:bg-blue-800" : ""
          }`}
          aria-current={isSelected ? "true" : "false"}
        >
          <img
            src={`https://placehold.co/40x40/4a90e2/ffffff/png?text=${user.username
              .slice(0, 2)
              .toUpperCase()}`}
            alt={`Avatar of user ${user.username}`}
            className="w-10 h-10 rounded-full object-cover"
            loading="lazy"
          />
          <span className="font-medium truncate">{user.username}</span>
        </button>
      );
    }

    // Chat window component
    function ChatWindow({ currentUser, chatUser, token, onLogout }) {
      const [messages, setMessages] = useState([]);
      const [input, setInput] = useState("");
      const [ws, setWs] = useState(null);
      const [status, setStatus] = useState("Connecting...");
      const messagesEndRef = useRef(null);
      const [unreadCount, setUnreadCount] = useState(0);
      const [darkMode, setDarkMode] = useState(
        localStorage.getItem("chatapp_darkmode") === "true"
      );

      // Scroll to bottom on new messages
      useEffect(() => {
        if (messagesEndRef.current) {
          messagesEndRef.current.scrollIntoView({ behavior: "smooth" });
        }
      }, [messages]);

      // Toggle dark mode
      useEffect(() => {
        if (darkMode) {
          document.documentElement.classList.add("dark");
        } else {
          document.documentElement.classList.remove("dark");
        }
        localStorage.setItem("chatapp_darkmode", darkMode);
      }, [darkMode]);

      // Fetch chat history with selected user
      useEffect(() => {
        if (!chatUser) return;
        async function fetchMessages() {
          try {
            const res = await fetch(
              API_BASE + `/api/messages/${chatUser.id}`,
              {
                headers: { Authorization: "Bearer " + token },
              }
            );
            if (!res.ok) throw new Error("Failed to load messages");
            const data = await res.json();
            setMessages(data.messages);
            setUnreadCount(0);
          } catch (err) {
            console.error(err);
          }
        }
        fetchMessages();
      }, [chatUser, token]);

      // Setup WebSocket connection
      useEffect(() => {
        if (!chatUser) return;

        let socket = new WebSocket(WS_URL + `?token=${token}`);

        socket.onopen = () => {
          setStatus("Connected");
          // Join private room with chatUser id
          socket.send(
            JSON.stringify({
              type: "join",
              toUserId: chatUser.id,
            })
          );
        };

        socket.onmessage = (event) => {
          try {
            const data = JSON.parse(event.data);
            if (data.type === "message" && data.fromUserId === chatUser.id) {
              setMessages((prev) => [...prev, data.message]);
              setUnreadCount((c) => c + 1);
              // Notification
              if (document.hidden) {
                if (Notification.permission === "granted") {
                  new Notification(`New message from ${chatUser.username}`, {
                    body: data.message.text,
                    icon: "https://placehold.co/64x64/4a90e2/ffffff/png?text=Chat",
                  });
                }
              }
            }
            if (data.type === "status") {
              setStatus(data.status);
            }
          } catch (e) {
            console.error("Invalid WS message", e);
          }
        };

        socket.onclose = () => {
          setStatus("Disconnected");
        };

        socket.onerror = () => {
          setStatus("Error");
        };

        setWs(socket);

        return () => {
          socket.close();
          setWs(null);
        };
      }, [chatUser, token]);

      // Request notification permission on mount
      useEffect(() => {
        if ("Notification" in window && Notification.permission !== "granted") {
          Notification.requestPermission();
        }
      }, []);

      // Send message handler
      async function sendMessage() {
        if (!input.trim() || !ws || ws.readyState !== WebSocket.OPEN) return;
        const msg = input.trim();
        const messageObj = {
          type: "message",
          toUserId: chatUser.id,
          text: msg,
        };
        ws.send(JSON.stringify(messageObj));
        // Optimistic UI update
        setMessages((prev) => [
          ...prev,
          {
            id: "temp-" + Date.now(),
            fromUserId: currentUser.id,
            toUserId: chatUser.id,
            text: msg,
            createdAt: new Date().toISOString(),
          },
        ]);
        setInput("");
      }

      // Handle enter key in textarea
      function handleKeyDown(e) {
        if (e.key === "Enter" && !e.shiftKey) {
          e.preventDefault();
          sendMessage();
        }
      }

      return (
        <div className="flex flex-col h-full">
          <header className="flex items-center justify-between bg-white dark:bg-gray-800 border-b border-gray-300 dark:border-gray-700 px-4 py-3">
            <div className="flex items-center space-x-3">
              <img
                src={`https://placehold.co/40x40/4a90e2/ffffff/png?text=${chatUser.username
                  .slice(0, 2)
                  .toUpperCase()}`}
                alt={`Avatar of user ${chatUser.username}`}
                className="w-10 h-10 rounded-full object-cover"
                loading="lazy"
              />
              <div>
                <h2 className="font-semibold text-lg">{chatUser.username}</h2>
                <p className="text-xs text-gray-500 dark:text-gray-400">{status}</p>
              </div>
            </div>
            <div className="flex items-center space-x-3">
              <button
                onClick={() => setDarkMode((d) => !d)}
                aria-label="Toggle dark mode"
                className="text-gray-600 dark:text-gray-300 hover:text-blue-600 dark:hover:text-blue-400 focus:outline-none focus:ring-2 focus:ring-blue-500 rounded"
                title="Toggle dark mode"
              >
                {darkMode ? (
                  <i className="fas fa-sun"></i>
                ) : (
                  <i className="fas fa-moon"></i>
                )}
              </button>
              <button
                onClick={onLogout}
                aria-label="Logout"
                className="text-red-600 hover:text-red-800 dark:hover:text-red-400 focus:outline-none focus:ring-2 focus:ring-red-500 rounded"
                title="Logout"
              >
                <i className="fas fa-sign-out-alt"></i>
              </button>
            </div>
          </header>
          <main className="flex flex-1 overflow-hidden">
            <aside className="w-64 bg-white dark:bg-gray-800 border-r border-gray-300 dark:border-gray-700 flex flex-col">
              <div className="px-4 py-3 border-b border-gray-300 dark:border-gray-700 font-semibold text-lg">
                Contacts
              </div>
              <UserList
                currentUser={currentUser}
                selectedUser={chatUser}
                token={token}
                onSelectUser={(user) => {
                  setUnreadCount(0);
                  setMessages([]);
                  setStatus("Connecting...");
                  setInput("");
                  setWs(null);
                  setSelectedUser(user);
                }}
              />
            </aside>
            <section className="flex flex-col flex-1 bg-gray-100 dark:bg-gray-900">
              <div className="flex-1 overflow-y-auto p-4 scrollbar-thin">
                {messages.length === 0 && (
                  <p className="text-center text-gray-500 dark:text-gray-400 mt-10 select-none">
                    No messages yet. Start the conversation!
                  </p>
                )}
                {messages.map((msg) => (
                  <MessageBubble
                    key={msg.id}
                    message={msg}
                    isOwn={msg.fromUserId === currentUser.id}
                  />
                ))}
                <div ref={messagesEndRef} />
              </div>
              <form
                onSubmit={(e) => {
                  e.preventDefault();
                  sendMessage();
                }}
                className="bg-white dark:bg-gray-800 border-t border-gray-300 dark:border-gray-700 p-3 flex items-center space-x-3"
              >
                <textarea
                  rows="1"
                  value={input}
                  onChange={(e) => setInput(e.target.value)}
                  onKeyDown={handleKeyDown}
                  placeholder="Type a message"
                  className="flex-1 resize-none rounded-md border border-gray-300 dark:border-gray-600 bg-gray-50 dark:bg-gray-700 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500 text-gray-900 dark:text-gray-100"
                  aria-label="Message input"
                  required
                />
                <button
                  type="submit"
                  disabled={!input.trim()}
                  className="bg-blue-600 hover:bg-blue-700 disabled:bg-blue-400 text-white rounded-md px-4 py-2 transition"
                  aria-label="Send message"
                >
                  <i className="fas fa-paper-plane"></i>
                </button>
              </form>
            </section>
          </main>
        </div>
      );
    }

    // User list component (fetches all users except current)
    function UserList({ currentUser, selectedUser, token, onSelectUser }) {
      const [users, setUsers] = useState([]);
      const [loading, setLoading] = useState(true);
      const [error, setError] = useState(null);

      useEffect(() => {
        async function fetchUsers() {
          setLoading(true);
          setError(null);
          try {
            const res = await fetch(API_BASE + "/api/users", {
              headers: { Authorization: "Bearer " + token },
            });
            if (!res.ok) throw new Error("Failed to load users");
            const data = await res.json();
            // Exclude current user
            const filtered = data.users.filter((u) => u.id !== currentUser.id);
            setUsers(filtered);
          } catch (err) {
            setError(err.message);
          } finally {
            setLoading(false);
          }
        }
        fetchUsers();
      }, [currentUser.id, token]);

      if (loading) {
        return (
          <div className="flex justify-center items-center flex-1 text-gray-500 dark:text-gray-400">
            Loading users...
          </div>
        );
      }
      if (error) {
        return (
          <div className="p-4 text-red-600 dark:text-red-400 font-medium">
            {error}
          </div>
        );
      }
      if (users.length === 0) {
        return (
          <div className="p-4 text-gray-500 dark:text-gray-400 select-none">
            No other users found.
          </div>
        );
      }

      return (
        <nav className="flex-1 overflow-y-auto">
          {users.map((user) => (
            <UserListItem
              key={user.id}
              user={user}
              isSelected={selectedUser && selectedUser.id === user.id}
              onClick={onSelectUser}
            />
          ))}
        </nav>
      );
    }

    // Main App component
    function App() {
      const [auth, setAuth] = useState(() => {
        try {
          const saved = localStorage.getItem(authStorageKey);
          if (!saved) return null;
          return JSON.parse(saved);
        } catch {
          return null;
        }
      });
      const [selectedUser, setSelectedUser] = useState(null);

      // Save auth to localStorage on change
      useEffect(() => {
        if (auth) {
          localStorage.setItem(authStorageKey, JSON.stringify(auth));
        } else {
          localStorage.removeItem(authStorageKey);
        }
      }, [auth]);

      // If logged out, clear selected user
      useEffect(() => {
        if (!auth) setSelectedUser(null);
      }, [auth]);

      // If no selected user, pick first user from user list after login
      // We do this by fetching users once after login
      useEffect(() => {
        if (!auth) return;
        async function fetchFirstUser() {
          try {
            const res = await fetch(API_BASE + "/api/users", {
              headers: { Authorization: "Bearer " + auth.token },
            });
            if (!res.ok) return;
            const data = await res.json();
            const others = data.users.filter((u) => u.id !== auth.user.id);
            if (others.length > 0) setSelectedUser(others[0]);
          } catch {}
        }
        fetchFirstUser();
      }, [auth]);

      function handleLogin(token, user) {
        setAuth({ token, user });
      }

      function handleLogout() {
        setAuth(null);
      }

      if (!auth) {
        return <Login onLogin={handleLogin} />;
      }

      if (!selectedUser) {
        return (
          <div className="flex flex-col justify-center items-center h-full px-4 text-gray-700 dark:text-gray-300">
            <p className="mb-4">Loading contacts...</p>
            <div className="animate-spin rounded-full h-8 w-8 border-t-2 border-b-2 border-blue-600"></div>
          </div>
        );
      }

      return (
        <ChatWindow
          currentUser={auth.user}
          chatUser={selectedUser}
          token={auth.token}
          onLogout={handleLogout}
          setSelectedUser={setSelectedUser}
        />
      );
    }

    ReactDOM.createRoot(document.getElementById("root")).render(<App />);
  </script>
</body>
</html>
