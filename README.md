/project
 ├─ /client        <- ton React app
 ├─ /config
 │    └─ db.js
 ├─ /routes
 │    ├─ authRoutes.js
 │    ├─ postRoutes.js
 │    ├─ commentRoutes.js
 │    └─ categoryRoutes.js
 ├─ /models
 │    └─ User.js
 └─ server.js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
    username: { type: String, required: true, unique: true },
    password: { type: String, required: true },
    avatar: { type: String, default: "" },
    role: { type: String, enum: ["user", "admin"], default: "user" },
    createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model("User", userSchema);
const mongoose = require("mongoose");

const connectDB = async () => {
    try {
        await mongoose.connect(process.env.MONGO_URI, {
            useNewUrlParser: true,
            useUnifiedTopology: true
        });
        console.log("MongoDB connecté ✅");
    } catch (error) {
        console.error("Erreur MongoDB :", error.message);
        process.exit(1);
    }
};

module.exports = connectDB;
require("dotenv").config();
const express = require("express");
const cors = require("cors");
const helmet = require("helmet");
const rateLimit = require("express-rate-limit");
const path = require("path");
const connectDB = require("./config/db");

const app = express();

// ----- Middleware -----
app.use(cors());
app.use(helmet());
app.use(express.json());
app.use(rateLimit({ windowMs: 60 * 1000, max: 100 }));

// ----- Connexion DB -----
connectDB();

// ----- Routes API -----
app.use("/api/auth", require("./routes/authRoutes"));
app.use("/api/posts", require("./routes/postRoutes"));
app.use("/api/comments", require("./routes/commentRoutes"));
app.use("/api/categories", require("./routes/categoryRoutes"));

// ----- Serve React build -----
app.use(express.static(path.join(__dirname, "client/build")));

app.get("*", (req, res) => {
    res.sendFile(path.resolve(__dirname, "client", "build", "index.html"));
});

// ----- Lancement serveur -----
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Serveur OK sur port ${PORT}`));
import { createContext, useState, useEffect } from "react";
import { api } from "../api";

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
    const [user, setUser] = useState(null);

    useEffect(() => {
        const saved = localStorage.getItem("user");
        if (saved) setUser(JSON.parse(saved));
    }, []);

    const login = async (username, password) => {
        const res = await api.post("/auth/login", { username, password });
        localStorage.setItem("token", res.data.token);
        localStorage.setItem("user", JSON.stringify(res.data.user));
        setUser(res.data.user);
    };

    const register = async (username, password) => {
        await api.post("/auth/register", { username, password });
        return login(username, password);
    };

    const logout = () => {
        setUser(null);
        localStorage.removeItem("token");
        localStorage.removeItem("user");
    };

    const isAdmin = () => user?.role === "admin";

    return (
        <AuthContext.Provider value={{ user, login, register, logout, isAdmin }}>
            {children}
        </AuthContext.Provider>
    );
};
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import { useContext } from "react";
import { AuthContext } from "./context/AuthContext";
import Navbar from "./components/Navbar";
import Home from "./pages/Home";
import Login from "./pages/Login";
import Register from "./pages/Register";
import Profile from "./pages/Profile";
import Post from "./pages/Post";
import AdminPanel from "./components/AdminPanel";

function ProtectedAdminRoute({ children }) {
    const { user, isAdmin } = useContext(AuthContext);
    
    if (!user || !isAdmin()) {
        return <Navigate to="/" replace />;
    }
    
    return children;
}

export default function App() {
    return (
        <BrowserRouter>
            <Navbar />
            <Routes>
                <Route path="/" element={<Home />} />
                <Route path="/login" element={<Login />} />
                <Route path="/register" element={<Register />} />
                <Route path="/profile/:username" element={<Profile />} />
                <Route path="/post/:id" element={<Post />} />
                <Route 
                    path="/admin" 
                    element={
                        <ProtectedAdminRoute>
                            <AdminPanel />
                        </ProtectedAdminRoute>
                    } 
                />
            </Routes>
        </BrowserRouter>
    );
}
