<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>NebulaFeed 2.0</title>

    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            background: #d9e8ff; /* bleu clair */
        }

        header {
            background: #003566; /* bleu foncé */
            color: white;
            padding: 15px;
            text-align: center;
            font-size: 26px;
            font-weight: bold;
        }

        .container {
            width: 700px;
            margin: 25px auto;
        }

        /* Auth */
        .auth-box {
            background: white;
            padding: 20px;
            border-radius: 8px;
            margin-bottom: 20px;
            text-align: center;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        }

        .auth-box input {
            width: 95%;
            padding: 10px;
            margin: 5px 0;
        }

        .auth-box button {
            padding: 10px 20px;
            background: #001d3d;
            color: white;
            border: none;
            margin-top: 10px;
            cursor: pointer;
            border-radius: 5px;
        }

        .post-form {
            background: white;
            padding: 15px;
            border-radius: 8px;
            margin-bottom: 20px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        }

        .post-form input,
        .post-form select,
        .post-form textarea {
            width: 100%;
            padding: 8px;
            margin-bottom: 10px;
        }

        .post-form button {
            background: #003566;
            color: white;
            padding: 10px;
            border: none;
            cursor: pointer;
            border-radius: 5px;
        }

        /* Filtre */
        .filter-box {
            margin-bottom: 20px;
            background: #fff;
            padding: 10px;
            border-radius: 8px;
        }

        /* Posts */
        .post {
            background: white;
            padding: 15px;
            border-radius: 8px;
            margin-bottom: 20px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        }

        .post img {
            max-width: 100%;
            margin-top: 10px;
            border-radius: 5px;
        }

        .category {
            padding: 4px 8px;
            background: #00aaff44; /* bleu clair transparent */
            border-radius: 5px;
            display: inline-block;
            margin-bottom: 10px;
            color: #003566;
            font-size: 12px;
        }

        .vote-box {
            margin-top: 10px;
            display: flex;
            gap: 10px;
            font-size: 22px;
            align-items: center;
        }

        .vote-btn {
            cursor: pointer;
            user-select: none;
        }

        /* COMMENTAIRES */
        .comment-section {
            background: #f1f6ff;
            padding: 10px;
            border-radius: 5px;
            margin-top: 15px;
        }

        .comment {
            padding: 8px;
            background: #fff;
            border-radius: 5px;
            margin-bottom: 5px;
        }

        .comment-form input {
            width: 90%;
            padding: 6px;
        }

        .comment-form button {
            padding: 6px 10px;
            background: #001d3d;
            color: white;
            border-radius: 4px;
            border: none;
            cursor: pointer;
        }

    </style>
</head>

<body>

<header>NebulaFeed 2.0</header>

<div class="container">

    <!-- LOGIN SYSTEM -->
    <div id="auth" class="auth-box">
        <h3>Connexion / Inscription</h3>

        <input type="text" id="username" placeholder="Nom d'utilisateur">
        <input type="password" id="password" placeholder="Mot de passe">

        <button onclick="login()">Connexion</button>
        <button onclick="register()">Créer un compte</button>
    </div>

    <div id="welcome" class="auth-box" style="display:none;">
        Connecté en tant que : <b id="userDisplay"></b>
        <button onclick="logout()">Déconnexion</button>
    </div>

    <!-- Formulaire -->
    <div id="postForm" class="post-form" style="display:none;">
        <input type="text" id="title" placeholder="Titre du post">
        <textarea id="content" placeholder="Contenu"></textarea>

        <select id="category">
            <option value="Général">Général</option>
            <option value="Meme">Meme</option>
            <option value="Gaming">Gaming</option>
            <option value="Technologie">Technologie</option>
            <option value="Art">Art</option>
        </select>

        <input type="file" id="imageFile" accept="image/*">

        <button onclick="addPost()">Publier</button>
    </div>

    <!-- Filtre -->
    <div class="filter-box">
        <select id="filter" onchange="displayPosts()">
            <option value="all">Toutes les catégories</option>
            <option value="Général">Général</option>
            <option value="Meme">Meme</option>
            <option value="Gaming">Gaming</option>
            <option value="Technologie">Technologie</option>
            <option value="Art">Art</option>
        </select>
    </div>

    <div id="posts"></div>

</div>

<script>
/* --------------------- SYSTEME UTILISATEUR --------------------- */

let currentUser = null;

function register() {
    let user = username.value;
    let pass = password.value;

    if (!user || !pass) return alert("Remplis tout !");

    if (localStorage.getItem("user_" + user))
        return alert("Ce nom est déjà utilisé.");

    localStorage.setItem("user_" + user, pass);
    alert("Compte créé !");
}

function login() {
    let user = username.value;
    let pass = password.value;

    if (localStorage.getItem("user_" + user) === pass) {
        currentUser = user;
        updateUI();
    } else {
        alert("Mauvais identifiants");
    }
}

function logout() {
    currentUser = null;
    updateUI();
}

function updateUI() {
    if (currentUser) {
        auth.style.display = "none";
        welcome.style.display = "block";
        userDisplay.innerText = currentUser;
        postForm.style.display = "block";
    } else {
        auth.style.display = "block";
        welcome.style.display = "none";
        postForm.style.display = "none";
    }

    displayPosts();
}

/* --------------------- POSTS --------------------- */

let posts = [];

function addPost() {
    if (!currentUser) return alert("Connecte-toi !");

    let file = imageFile.files[0];
    let imgURL = file ? URL.createObjectURL(file) : "";

    let post = {
        id: Date.now(),
        user: currentUser,
        title: title.value,
        content: content.value,
        category: category.value,
        image: imgURL,
        votes: 0,
        comments: []
    };

    posts.unshift(post);
    displayPosts();
}

function vote(id, value) {
    let post = posts.find(p => p.id === id);
    post.votes += value;
    displayPosts();
}

/* --------------------- COMMENTAIRES --------------------- */

function addComment(id) {
    let input = document.getElementById("comment-input-" + id);
    let text = input.value;

    if (!text) return;

    let post = posts.find(p => p.id === id);
    post.comments.push({ user: currentUser, text });

    input.value = "";
    displayPosts();
}

/* --------------------- AFFICHAGE --------------------- */

function displayPosts() {
    const filter = document.getElementById("filter").value;

    postsDiv = document.getElementById("posts");
    postsDiv.innerHTML = "";

    posts
        .filter(p => filter === "all" || p.category === filter)
        .forEach(p => {

            let post = document.createElement("div");
            post.className = "post";

            post.innerHTML = `
                <div class="category">${p.category}</div>
                <h3>${p.title}</h3>
                <i>par ${p.user}</i>
                <p>${p.content}</p>
                ${p.image ? `<img src="${p.image}">` : ""}

                <div class="vote-box">
                    <span class="vote-btn" onclick="vote(${p.id}, 1)">⬆️</span>
                    <span><b>${p.votes}</b></span>
                    <span class="vote-btn" onclick="vote(${p.id}, -1)">⬇️</span>
                </div>

                <div class="comment-section">
                    <h4>Commentaires</h4>
                    ${p.comments.map(c => `
                        <div class="comment"><b>${c.user}</b> : ${c.text}</div>
                    `).join("")}

                    <div class="comment-form">
                        <input id="comment-input-${p.id}" placeholder="Écrire un commentaire...">
                        <button onclick="addComment(${p.id})">Envoyer</button>
                    </div>
                </div>
            `;

            postsDiv.append(post);
        });
}

</script>

</body>
</html>
