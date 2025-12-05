nebula-feed/
│
├── backend/
│   ├── server.js
│   ├── config/
│   │   ├── db.js
│   │   └── cloudinary.js
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── postController.js
│   │   └── commentController.js
│   ├── middleware/
│   │   ├── authMiddleware.js
│   │   └── upload.js
│   ├── models/
│   │   ├── User.js
│   │   ├── Post.js
│   │   └── Comment.js
│   └── routes/
│       ├── authRoutes.js
│       ├── postRoutes.js
│       └── commentRoutes.js
│
└── frontend/
    ├── package.json
    ├── public/
    │   └── index.html
    └── src/
        ├── App.jsx
        ├── index.js
        ├── api.js
        ├── styles.css
        ├── context/
        │   ├── AuthContext.jsx
        │   └── ThemeContext.jsx
        ├── components/
        │   ├── Navbar.jsx
        │   ├── PostCard.jsx
        │   ├── CreatePost.jsx
        │   ├── CommentSection.jsx
        │   └── ThemeToggle.jsx
        └── pages/
            ├── Home.jsx
            ├── Login.jsx
            ├── Register.jsx
            ├── Profile.jsx
            └── Post.jsx
require("dotenv").config();
const express = require("express");
const cors = require("cors");
const helmet = require("helmet");
const rateLimit = require("express-rate-limit");
const connectDB = require("./config/db");

const app = express();

// Sécurité
app.use(cors());
app.use(helmet());
app.use(express.json());
app.use(rateLimit({ windowMs: 60 * 1000, max: 100 }));

// Connexion DB
connectDB();

// Routes
app.use("/api/auth", require("./routes/authRoutes"));
app.use("/api/posts", require("./routes/postRoutes"));
app.use("/api/comments", require("./routes/commentRoutes"));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Serveur OK sur port ${PORT}`));
const mongoose = require("mongoose");

const connectDB = async () => {
    try {
        await mongoose.connect(process.env.MONGO_URI);
        console.log("MongoDB connecté ✔");
    } catch (err) {
        console.error(err);
        process.exit(1);
    }
};

module.exports = connectDB;
const cloudinary = require("cloudinary").v2;

cloudinary.config({
    cloud_name: process.env.CLOUD_NAME,
    api_key: process.env.CLOUD_KEY,
    api_secret: process.env.CLOUD_SECRET,
});

module.exports = cloudinary;
const jwt = require("jsonwebtoken");

module.exports = (req, res, next) => {
    const token = req.header("Authorization")?.split(" ")[1];
    if (!token) return res.status(401).json({ message: "Non autorisé" });

    try {
        const user = jwt.verify(token, process.env.JWT_SECRET);
        req.user = user;
        next();
    } catch {
        res.status(401).json({ message: "Token invalide" });
    }
};
const multer = require("multer");

const storage = multer.memoryStorage();
module.exports = multer({ storage });
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
    username: { type: String, required: true, unique: true },
    password: { type: String, required: true },
    avatar: { type: String, default: "" },
});

module.exports = mongoose.model("User", userSchema);
const mongoose = require("mongoose");

const postSchema = new mongoose.Schema({
    author: { type: mongoose.Schema.Types.ObjectId, ref: "User" },
    title: String,
    content: String,
    image: String,
    category: String,
    votes: { type: Number, default: 0 },
    createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model("Post", postSchema);
const mongoose = require("mongoose");

const commentSchema = new mongoose.Schema({
    postId: { type: mongoose.Schema.Types.ObjectId, ref: "Post" },
    author: { type: mongoose.Schema.Types.ObjectId, ref: "User" },
    content: String,
    createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model("Comment", commentSchema);
const User = require("../models/User");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");

exports.register = async (req, res) => {
    const { username, password } = req.body;
    const hashed = await bcrypt.hash(password, 10);
    const user = await User.create({ username, password: hashed });
    res.json({ message: "Compte créé", user });
};

exports.login = async (req, res) => {
    const { username, password } = req.body;
    const user = await User.findOne({ username });
    if (!user) return res.status(404).json({ message: "Utilisateur non trouvé" });

    const valid = await bcrypt.compare(password, user.password);
    if (!valid) return res.status(400).json({ message: "Mot de passe incorrect" });

    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET);
    res.json({ token, user });
};
const Post = require("../models/Post");
const cloudinary = require("../config/cloudinary");

exports.createPost = async (req, res) => {
    let imageUrl = "";

    if (req.file) {
        const result = await cloudinary.uploader.upload_stream({ resource_type: "image" }, (err, resu) => {});
        const stream = cloudinary.uploader.upload_stream({ resource_type: "image" }, (err, result) => {
            if (result) imageUrl = result.secure_url;
        });
        stream.end(req.file.buffer);
    }

    const post = await Post.create({
        author: req.user.id,
        title: req.body.title,
        content: req.body.content,
        category: req.body.category,
        image: imageUrl
    });

    res.json(post);
};

exports.getPosts = async (req, res) => {
    const posts = await Post.find().populate("author", "username avatar");
    res.json(posts);
};

exports.vote = async (req, res) => {
    const post = await Post.findById(req.params.id);
    post.votes += req.body.value;
    await post.save();
    res.json(post);
};
const router = require("express").Router();
const { register, login } = require("../controllers/authController");

router.post("/register", register);
router.post("/login", login);

module.exports = router;
const router = require("express").Router();
const auth = require("../middleware/authMiddleware");
const upload = require("../middleware/upload");
const { createPost, getPosts, vote } = require("../controllers/postController");

router.get("/", getPosts);
router.post("/", auth, upload.single("image"), createPost);
router.post("/:id/vote", auth, vote);

module.exports = router;
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import { AuthProvider } from "./context/AuthContext";
import { ThemeProvider } from "./context/ThemeContext";
import "./styles.css";

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
    <React.StrictMode>
        <ThemeProvider>
            <AuthProvider>
                <App />
            </AuthProvider>
        </ThemeProvider>
    </React.StrictMode>
);
import { BrowserRouter, Routes, Route } from "react-router-dom";
import Navbar from "./components/Navbar";
import Home from "./pages/Home";
import Login from "./pages/Login";
import Register from "./pages/Register";
import Profile from "./pages/Profile";
import Post from "./pages/Post";

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
            </Routes>
        </BrowserRouter>
    );
}
import axios from "axios";

export const api = axios.create({
    baseURL: "http://localhost:5000/api",
});

api.interceptors.request.use((config) => {
    const token = localStorage.getItem("token");
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
});
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

    return (
        <AuthContext.Provider value={{ user, login, register, logout }}>
            {children}
        </AuthContext.Provider>
    );
};
import { createContext, useState } from "react";

export const ThemeContext = createContext();

export const ThemeProvider = ({ children }) => {
    const [dark, setDark] = useState(true);

    const toggleTheme = () => setDark(!dark);

    return (
        <ThemeContext.Provider value={{ dark, toggleTheme }}>
            <div className={dark ? "dark" : "light"}>
                {children}
            </div>
        </ThemeContext.Provider>
    );
};
import { useContext } from "react";
import { Link } from "react-router-dom";
import { AuthContext } from "../context/AuthContext";
import ThemeToggle from "./ThemeToggle";

export default function Navbar() {
    const { user, logout } = useContext(AuthContext);

    return (
        <nav className="navbar">
            <Link to="/" className="logo">NebulaFeed</Link>

            <div className="nav-right">
                <ThemeToggle />

                {!user ? (
                    <>
                        <Link to="/login">Connexion</Link>
                        <Link to="/register">Inscription</Link>
                    </>
                ) : (
                    <>
                        <Link to={`/profile/${user.username}`}>{user.username}</Link>
                        <button onClick={logout} className="logout">Déconnexion</button>
                    </>
                )}
            </div>
        </nav>
    );
}
import { useContext } from "react";
import { ThemeContext } from "../context/ThemeContext";

export default function ThemeToggle() {
    const { dark, toggleTheme } = useContext(ThemeContext);
    return (
        <button onClick={toggleTheme} className="theme-toggle">
            {dark ? "☀️" : "🌙"}
        </button>
    );
}
import { motion } from "framer-motion";
import { api } from "../api";
import { useContext } from "react";
import { AuthContext } from "../context/AuthContext";
import { Link } from "react-router-dom";

export default function PostCard({ post }) {
    const { user } = useContext(AuthContext);

    const vote = async (v) => {
        await api.post(`/posts/${post._id}/vote`, { value: v });
        window.location.reload();
    };

    return (
        <motion.div className="post-card" whileHover={{ scale: 1.02 }}>
            <div className="post-header">
                <b>{post.author.username}</b>
                <span className="category">{post.category}</span>
            </div>

            <Link to={`/post/${post._id}`}>
                <h3>{post.title}</h3>
                <p>{post.content}</p>
                {post.image && <img src={post.image} className="post-img" alt="" />}
            </Link>

            <div className="post-footer">
                <div className="votes">
                    <button onClick={() => vote(1)} className="vote-btn up">▲</button>
                    <span>{post.votes}</span>
                    <button onClick={() => vote(-1)} className="vote-btn down">▼</button>
                </div>
            </div>
        </motion.div>
    );
}
import { useState, useContext } from "react";
import { api } from "../api";
import { AuthContext } from "../context/AuthContext";

export default function CreatePost({ onPostCreated }) {
    const { user } = useContext(AuthContext);
    const [title, setTitle] = useState("");
    const [content, setContent] = useState("");
    const [category, setCategory] = useState("Meme");
    const [file, setFile] = useState(null);

    const submit = async (e) => {
        e.preventDefault();
        const formData = new FormData();
        formData.append("title", title);
        formData.append("content", content);
        formData.append("category", category);
        if (file) formData.append("image", file);

        await api.post("/posts", formData);
        setTitle(""); setContent(""); setCategory("Meme"); setFile(null);
        if (onPostCreated) onPostCreated();
    };

    if (!user) return null;

    return (
        <form onSubmit={submit} className="create-post">
            <input value={title} onChange={e => setTitle(e.target.value)} placeholder="Titre" required />
            <textarea value={content} onChange={e => setContent(e.target.value)} placeholder="Contenu..." required />
            <select value={category} onChange={e => setCategory(e.target.value)}>
                <option value="Meme">Meme</option>
                <option value="Gaming">Gaming</option>
                <option value="Technologie">Technologie</option>
                <option value="Art">Art</option>
                <option value="Actualités">Actualités</option>
            </select>
            <input type="file" onChange={e => setFile(e.target.files[0])} />
            <button type="submit">Publier</button>
        </form>
    );
}
import { useEffect, useState } from "react";
import { api } from "../api";
import PostCard from "../components/PostCard";
import CreatePost from "../components/CreatePost";

export default function Home() {
    const [posts, setPosts] = useState([]);
    const [filter, setFilter] = useState("all");

    const load = async () => {
        const res = await api.get("/posts");
        const all = res.data;
        if (filter === "all") setPosts(all);
        else setPosts(all.filter(p => p.category === filter));
    };

    useEffect(() => {
        load();
    }, [filter]);

    return (
        <div className="home">
            <CreatePost onPostCreated={load} />
            <div className="category-filter">
                <select onChange={e => setFilter(e.target.value)} value={filter}>
                    <option value="all">Toutes</option>
                    <option value="Meme">Meme</option>
                    <option value="Gaming">Gaming</option>
                    <option value="Technologie">Technologie</option>
                    <option value="Art">Art</option>
                    <option value="Actualités">Actualités</option>
                </select>
            </div>

            {posts.map(post => (
                <PostCard key={post._id} post={post} />
            ))}
        </div>
    );
}
import { useState, useContext } from "react";
import { AuthContext } from "../context/AuthContext";
import { useNavigate } from "react-router-dom";

export default function Login() {
    const { login } = useContext(AuthContext);
    const [username, setUsername] = useState("");
    const [password, setPassword] = useState("");
    const navigate = useNavigate();

    const submit = async (e) => {
        e.preventDefault();
        try {
            await login(username, password);
            navigate("/");
        } catch (err) {
            alert("Erreur de connexion");
        }
    };

    return (
        <form onSubmit={submit} className="auth-form">
            <h2>Connexion</h2>
            <input placeholder="Nom d'utilisateur" value={username} onChange={e => setUsername(e.target.value)} required />
            <input placeholder="Mot de passe" type="password" value={password} onChange={e => setPassword(e.target.value)} required />
            <button type="submit">Se connecter</button>
        </form>
    );
}
import { useState, useContext } from "react";
import { AuthContext } from "../context/AuthContext";
import { useNavigate } from "react-router-dom";

export default function Register() {
    const { register } = useContext(AuthContext);
    const [username, setUsername] = useState("");
    const [password, setPassword] = useState("");
    const navigate = useNavigate();

    const submit = async (e) => {
        e.preventDefault();
        try {
            await register(username, password);
            navigate("/");
        } catch (err) {
            alert("Erreur d'inscription");
        }
    };

    return (
        <form onSubmit={submit} className="auth-form">
            <h2>Inscription</h2>
            <input placeholder="Nom d'utilisateur" value={username} onChange={e => setUsername(e.target.value)} required />
            <input placeholder="Mot de passe" type="password" value={password} onChange={e => setPassword(e.target.value)} required />
            <button type="submit">S'inscrire</button>
        </form>
    );
}
import { useParams } from "react-router-dom";
import { useEffect, useState } from "react";
import { api } from "../api";
import PostCard from "../components/PostCard";

export default function Profile() {
    const { username } = useParams();
    const [posts, setPosts] = useState([]);

    useEffect(() => {
        const load = async () => {
            const res = await api.get("/posts");
            setPosts(res.data.filter(p => p.author.username === username));
        };
        load();
    }, [username]);

    return (
        <div className="profile">
            <h2>Profil : {username}</h2>
            {posts.map(post => (
                <PostCard key={post._id} post={post} />
            ))}
        </div>
    );
}
import { useParams } from "react-router-dom";
import { useEffect, useState } from "react";
import { api } from "../api";
import CommentSection from "../components/CommentSection";

export default function Post() {
    const { id } = useParams();
    const [post, setPost] = useState(null);

    useEffect(() => {
        const load = async () => {
            const res = await api.get("/posts");
            const p = res.data.find(p => p._id === id);
            setPost(p);
        };
        load();
    }, [id]);

    if (!post) return <div>Chargement...</div>;

    return (
        <div className="single-post">
            <h2>{post.title}</h2>
            <p>{post.content}</p>
            {post.image && <img src={post.image} alt="" />}
            <CommentSection postId={post._id} />
        </div>
    );
}
import { useEffect, useState, useContext } from "react";
import { api } from "../api";
import { AuthContext } from "../context/AuthContext";

export default function CommentSection({ postId }) {
    const [comments, setComments] = useState([]);
    const [content, setContent] = useState("");
    const { user } = useContext(AuthContext);

    const load = async () => {
        const res = await api.get("/comments?postId=" + postId);
        setComments(res.data);
    };

    useEffect(() => { load(); }, [postId]);

    const submit = async (e) => {
        e.preventDefault();
        if (!user) return alert("Connectez-vous pour commenter");
        await api.post("/comments", { postId, content });
        setContent("");
        load();
    };

    return (
        <div className="comments">
            <h4>Commentaires</h4>
            {comments.map(c => (
                <div key={c._id} className="comment">
                    <b>{c.author.username}</b> : {c.content}
                </div>
            ))}

            <form onSubmit={submit} className="comment-form">
                <input placeholder="Ajouter un commentaire..." value={content} onChange={e => setContent(e.target.value)} />
                <button type="submit">Envoyer</button>
            </form>
        </div>
    );
}
body.dark { background: #0b1e33; color: #fff; }
body.light { background: #f3f6fc; color: #111; }

.navbar { background: #07192b; padding: 15px 30px; display: flex; justify-content: space-between; color: white; }
.logo { font-size: 26px; font-weight: bold; }
.nav-right a, .logout, .theme-toggle { margin-left: 10px; color: #fff; text-decoration: none; }
.logout { background: none; border: none; cursor: pointer; }
.theme-toggle { background: none; border: none; cursor: pointer; font-size: 18px; }

.home, .profile, .single-post { max-width: 700px; margin: 20px auto; padding: 10px; }

.post-card { background: #11263d; padding: 20px; margin: 20px 0; border-radius: 12px; box-shadow: 0 4px 15px #00000044; }
.post-card h3 { margin-top: 0; }
.post-img { width: 100%; border-radius: 10px; margin-top: 10px; }
.post-footer { display: flex; justify-content: flex-start; align-items: center; margin-top: 10px; }
.vote-btn { margin: 0 5px; cursor: pointer; font-size: 18px; transition: 0.15s; }
.vote-btn:hover { transform: scale(1.3); }
.vote-btn.up { color: #4aa8ff; }
.vote-btn.down { color: #ff5e5e; }
.category { background: #4aa8ff33; padding: 4px 10px; border-radius: 5px; margin-left: 10px; }

.auth-form, .create-post, .comment-form { display: flex; flex-direction: column; margin: 20px 0; }
.auth-form input, .create-post input, .create-post textarea, .comment-form input, .auth-form select, .create-post select { margin: 5px 0; padding: 8px; border-radius: 5px; border: none; }
.auth-form button, .create-post button, .comment-form button { margin-top: 10px; padding: 8px; border-radius: 5px; cursor: pointer; background: #4aa8ff; color: white; border: none; }
.category-filter { margin: 10px 0; }
.comments { margin-top: 20px; }
.comment { margin: 5px 0; }
