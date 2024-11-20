# Documentation de l'Application Shopify Image Generator pour Poche et Fils

Salut Jeff,

Voici la documentation pour notre **Application Shopify Image Generator** destinée à **Poche et Fils**. Cette application utilise notre script Python existant avec l'API StabilityAI pour générer et traiter des images, les transformant en un format élégant de poche. Les images seront ensuite affichées directement sur le site Shopify de Poche et Fils.

Ce guide te fournira une vue d'ensemble des étapes nécessaires pour créer, développer et déployer l'application Shopify. Allons-y !

---

## Table des Matières


1. [Configurer l’Environnement de Développement](#configurer-lenvironnement-de-développement)
    - [Choisir un Framework Web en Python](#choisir-un-framework-web-en-python)
    - [Installer les Outils Nécessaires](#installer-les-outils-nécessaires)
2. [Développer le Backend avec Flask](#développer-le-backend-avec-flask)
    - [Créer une Application Flask](#créer-une-application-flask)
    - [Intégrer le Script de Génération d’Images](#intégrer-le-script-de-génération-dimages)
    - [Créer des Endpoints API](#créer-des-endpoints-api)
3. [Développer le Frontend avec React et Shopify Polaris](#développer-le-frontend-avec-react-et-shopify-polaris)
    - [Initialiser une Application React](#initialiser-une-application-react)
    - [Installer Polaris et Axios](#installer-polaris-et-axios)
    - [Construire l’Interface Utilisateur](#construire-linterface-utilisateur)
    - [Connecter le Frontend au Backend](#connecter-le-frontend-au-backend)
4. [Implémenter l’Authentification OAuth avec Shopify](#implémenter-lauthentification-oauth-avec-shopify)
    - [Configurer les Routes OAuth dans Flask](#configurer-les-routes-oauth-dans-flask)
    - [Configurer les Paramètres de l’App Shopify](#configurer-les-paramètres-de-lapp-shopify)
5. [Déployer l’Application sur DigitalOcean](#déployer-lapplication-sur-digitalocean)
    - [Préparer le Serveur sur DigitalOcean](#préparer-le-serveur-sur-digitalocean)
    - [Déployer le Backend Flask](#déployer-le-backend-flask)
    - [Déployer le Frontend React](#déployer-le-frontend-react)
6. [Tester et Finaliser l’App](#tester-et-finaliser-lapp)
    - [Tests Locaux](#tests-locaux)
    - [Tests en Production](#tests-en-production)
    - [Soumettre à l’App Store Shopify](#soumettre-à-lapp-store-shopify)
7. [Ressources Supplémentaires](#ressources-supplémentaires)

---


## Configurer l’Environnement de Développement

### Choisir un Framework Web en Python

Nous utiliserons **Flask** pour sa simplicité et sa flexibilité, idéal pour le backend de notre application.

### Installer les Outils Nécessaires

1. **Installer Python et Pip** : Assure-toi que Python 3.x et pip sont installés sur ta machine.
2. **Créer un Environnement Virtuel** :
    ```bash
    python3 -m venv venv
    source venv/bin/activate  # Sur Windows : venv\Scripts\activate
    ```
3. **Installer les Dépendances** :
    ```bash
    pip install Flask requests pillow python-dotenv gunicorn
    ```

---

## Développer le Backend avec Flask

### Créer une Application Flask

Crée un fichier nommé `app.py` et configure une application Flask de base.

```python
from flask import Flask, request, jsonify, send_file, redirect, url_for
import os
import requests
from PIL import Image, ImageDraw
from dotenv import load_dotenv
from urllib.parse import urlencode

load_dotenv()
STABILITY_API_KEY = os.getenv("STABILITY_API_KEY")
SHOPIFY_API_KEY = os.getenv("SHOPIFY_API_KEY")
SHOPIFY_API_SECRET = os.getenv("SHOPIFY_API_SECRET")
REDIRECT_URI = os.getenv("REDIRECT_URI")  # Exemple : "https://pocheetfils.digitalocean.app/auth/callback"

app = Flask(__name__)

# Template de prompt global
GLOBAL_PROMPT_TEMPLATE = (
    "Create a {image_type} with a {color_scheme} color scheme based on the following description: \"{user_prompt}\". "
    "The {image_type} should be simple, modern, and easily recognizable."
)

def generate_custom_image(api_key, user_prompt, output_path, aspect_ratio="1:1", output_format="png", 
                         color_scheme="black and white", image_type="logo"):
    headers = {
        "authorization": f"Bearer {api_key}",
        "Accept": "image/*, application/json",
    }

    global_prompt = GLOBAL_PROMPT_TEMPLATE.format(
        image_type=image_type,
        color_scheme=color_scheme,
        user_prompt=user_prompt
    )

    files = {
        "prompt": (None, global_prompt),
        "aspect_ratio": (None, aspect_ratio),
        "output_format": (None, output_format),
    }

    try:
        response = requests.post("https://api.stability.ai/v2beta/stable-image/generate/ultra", headers=headers, files=files, stream=True)
        
        if response.status_code == 200:
            content_type = response.headers.get('Content-Type', '')
            if 'image' in content_type:
                with open(output_path, "wb") as f:
                    for chunk in response.iter_content(chunk_size=8192):
                        f.write(chunk)
                return True
            elif 'application/json' in content_type:
                response_data = response.json()
                image_url = response_data.get("image_url")
                if image_url:
                    img_response = requests.get(image_url, stream=True)
                    if img_response.status_code == 200:
                        with open(output_path, "wb") as f:
                            for chunk in img_response.iter_content(chunk_size=8192):
                                f.write(chunk)
                        return True
                    else:
                        return False
                else:
                    return False
            else:
                return False
        else:
            return False
    except Exception as e:
        print(f"Request error: {e}")
        return False

def apply_pocket_shape(image_path, output_path, pocket_size=(700, 1124)):
    image = Image.open(image_path).resize(pocket_size, Image.LANCZOS)
    
    mask = Image.new("L", pocket_size, 0)
    draw = ImageDraw.Draw(mask)
    
    width, height = pocket_size
    draw.polygon([
        (0, 0),
        (width, 0),
        (width, height - 200),
        (width // 2, height),
        (0, height - 200)
    ], fill=255)
    
    shaped_image = Image.new("RGBA", pocket_size)
    shaped_image.paste(image, (0, 0), mask)
    shaped_image.save(output_path, "PNG")
    return output_path

@app.route('/generate-images', methods=['POST'])
def generate_images():
    data = request.json
    user_prompt = data.get('user_prompt', '')
    num_images = data.get('num_images', 4)
    color_scheme = data.get('color_scheme', 'black and white')
    image_type = data.get('image_type', 'logo')
    aspect_ratio = data.get('aspect_ratio', '9:16')
    pocket_size = tuple(data.get('pocket_size', [700, 1124]))
    
    generated_image_paths = []
    final_image_paths = []
    
    for i in range(num_images):
        generated_path = f"static/generated_image_{i+1}.png"
        final_path = f"static/final_image_{i+1}.png"
        
        success = generate_custom_image(
            STABILITY_API_KEY,
            user_prompt,
            generated_path,
            aspect_ratio=aspect_ratio,
            color_scheme=color_scheme,
            image_type=image_type
        )
        
        if success:
            apply_pocket_shape(generated_path, final_path, pocket_size=pocket_size)
            generated_image_paths.append(generated_path)
            final_image_paths.append(final_path)
        else:
            return jsonify({"error": f"Échec de la génération de l'image {i+1}"}), 500
    
    return jsonify({"images": final_image_paths}), 200

@app.route('/images/<filename>', methods=['GET'])
def get_image(filename):
    return send_file(os.path.join('static', filename), mimetype='image/png')

@app.route('/auth', methods=['GET'])
def auth():
    shop = request.args.get('shop')
    if shop:
        params = {
            "client_id": SHOPIFY_API_KEY,
            "scope": "read_products,write_products",  # Ajuster les scopes selon les besoins
            "redirect_uri": REDIRECT_URI,
            "state": "random_string",  # Implémenter un état aléatoire sécurisé
            "grant_options[]": "per-user"
        }
        auth_url = f"https://{shop}/admin/oauth/authorize?" + urlencode(params)
        return redirect(auth_url)
    return "Paramètre shop manquant", 400

@app.route('/auth/callback', methods=['GET'])
def auth_callback():
    shop = request.args.get('shop')
    code = request.args.get('code')
    state = request.args.get('state')

    # Échanger le code contre un access token
    token_url = f"https://{shop}/admin/oauth/access_token"
    data = {
        "client_id": SHOPIFY_API_KEY,
        "client_secret": SHOPIFY_API_SECRET,
        "code": code
    }
    response = requests.post(token_url, data=data)
    response_data = response.json()
    access_token = response_data.get('access_token')

    if access_token:
        # TODO : Sauvegarder l'access token de manière sécurisée (par exemple, dans une base de données)
        return "Authentification réussie"
    else:
        return "Échec de l'authentification", 400

if __name__ == "__main__":
    if not STABILITY_API_KEY:
        raise ValueError("Clé API StabilityAI non trouvée dans les variables d'environnement.")
    if not os.path.exists('static'):
        os.makedirs('static')
    app.run(host='0.0.0.0', port=5000, debug=True)

requirements.txt

Gèle tes dépendances.

Flask==2.0.3
requests==2.27.1
Pillow==9.0.1
python-dotenv==0.20.0
gunicorn==20.1.0

Procfile

Définit le type de processus pour le déploiement sur DigitalOcean.

web: gunicorn app:app

.env

Stocke tes variables d’environnement de manière sécurisée.

STABILITY_API_KEY=ta_clé_api_stability_ai
SHOPIFY_API_KEY=ta_clé_api_shopify
SHOPIFY_API_SECRET=ton_secret_api_shopify
REDIRECT_URI=https://pocheetfils.digitalocean.app/auth/callback

Créer le Répertoire Static

Assure-toi que le dossier static existe pour stocker les images générées.

mkdir static

Développer le Frontend avec React et Shopify Polaris

Shopify utilise principalement React avec Polaris (leur bibliothèque de composants UI) pour le développement frontend des applications. Nous allons créer une application React qui interagit avec notre backend Flask pour générer et afficher les images.

Initialiser une Application React

	1.	Créer une Application React :

npx create-react-app shopify-app-frontend
cd shopify-app-frontend



Installer Polaris et Axios

npm install @shopify/polaris @shopify/app-bridge-react axios

Construire l’Interface Utilisateur

	1.	Modifier src/App.js :

import React, { useState } from 'react';
import '@shopify/polaris/dist/styles.css';
import { AppProvider, Page, Card, TextField, Button, Stack, Image, DisplayText } from '@shopify/polaris';
import axios from 'axios';

function App() {
  const [userPrompt, setUserPrompt] = useState('');
  const [images, setImages] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleGenerate = async () => {
    if (!userPrompt.trim()) {
      setError('Veuillez entrer une description pour l\'image.');
      return;
    }

    setLoading(true);
    setError('');
    setImages([]);

    try {
      const response = await axios.post('/generate-images', {  // Proxy configuré plus tard
        user_prompt: userPrompt,
        num_images: 4,
        color_scheme: 'black and white',
        image_type: 'logo',
        aspect_ratio: '9:16',
        pocket_size: [700, 1124]
      });

      if (response.data && response.data.images) {
        setImages(response.data.images);
      }
    } catch (err) {
      console.error(err);
      setError('Échec de la génération des images.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <AppProvider>
      <Page title="Application Générateur d'Images">
        <Card sectioned>
          <Stack vertical>
            <TextField
              label="Décrivez votre image"
              value={userPrompt}
              onChange={(value) => setUserPrompt(value)}
              placeholder="Exemple : Phare sur une falaise surplombant l'océan"
            />
            <Button onClick={handleGenerate} primary loading={loading}>
              Générer les Images
            </Button>
            {error && <DisplayText size="small" variation="negative">{error}</DisplayText>}
          </Stack>
        </Card>
        {images.length > 0 && (
          <Card title="Images Générées" sectioned>
            <Stack wrap>
              {images.map((imgPath, index) => (
                <Image
                  key={index}
                  source={`/${imgPath}`}  // Ajuster en fonction de l'URL backend
                  alt={`Image Générée ${index + 1}`}
                  width="200px"
                />
              ))}
            </Stack>
          </Card>
        )}
      </Page>
    </AppProvider>
  );
}

export default App;



Connecter le Frontend au Backend

Assure-toi que l’application React communique correctement avec le backend Flask. Lors du déploiement, mets à jour l’URL de la requête Axios pour pointer vers le backend en production.

Configurer le Proxy pour le Développement

Pour éviter les problèmes de CORS pendant le développement, configure un proxy dans package.json de ton application React :

{
  // ... autres configurations
  "proxy": "http://localhost:5000"
}

Cela permet à Axios d’envoyer des requêtes à /generate-images sans spécifier l’URL complète du backend.

Implémenter l’Authentification OAuth avec Shopify

Pour que l’application Shopify puisse accéder aux données d’une boutique, il est nécessaire d’implémenter l’authentification OAuth. Voici comment procéder :

Configurer les Routes OAuth dans Flask

Ajoute les routes nécessaires pour gérer le flux OAuth de Shopify dans app.py.

@app.route('/auth', methods=['GET'])
def auth():
    shop = request.args.get('shop')
    if shop:
        params = {
            "client_id": SHOPIFY_API_KEY,
            "scope": "read_products,write_products",  # Ajuster les scopes selon les besoins
            "redirect_uri": REDIRECT_URI,
            "state": "random_string",  # Implémenter un état aléatoire sécurisé
            "grant_options[]": "per-user"
        }
        auth_url = f"https://{shop}/admin/oauth/authorize?" + urlencode(params)
        return redirect(auth_url)
    return "Paramètre shop manquant", 400

@app.route('/auth/callback', methods=['GET'])
def auth_callback():
    shop = request.args.get('shop')
    code = request.args.get('code')
    state = request.args.get('state')

    # Échanger le code contre un access token
    token_url = f"https://{shop}/admin/oauth/access_token"
    data = {
        "client_id": SHOPIFY_API_KEY,
        "client_secret": SHOPIFY_API_SECRET,
        "code": code
    }
    response = requests.post(token_url, data=data)
    response_data = response.json()
    access_token = response_data.get('access_token')

    if access_token:
        # TODO : Sauvegarder l'access token de manière sécurisée (par exemple, dans une base de données)
        return "Authentification réussie"
    else:
        return "Échec de l'authentification", 400

Configurer les Paramètres de l’App Shopify

	1.	Définir les URLs de Redirection :
	•	Dans ton tableau de bord Shopify Partner, navigue vers les paramètres de ton application.
	•	Assure-toi que l’URL de redirection correspond à REDIRECT_URI dans ton fichier .env (exemple : https://pocheetfils.digitalocean.app/auth/callback).
	2.	Définir les Scopes :
	•	Ajuste le paramètre scope dans la route OAuth pour demander les permissions nécessaires.

Sécuriser le Flux OAuth

	•	Paramètre d’État (State) : Utilise un jeton aléatoire sécurisé pour protéger contre les attaques CSRF.
	•	Stocker les Access Tokens : Utilise une méthode sécurisée (comme une base de données) pour stocker les tokens d’accès Shopify.

Déployer l’Application sur DigitalOcean

Pour rendre notre application accessible depuis Shopify, elle doit être hébergée sur un serveur accessible publiquement avec un certificat SSL valide (HTTPS).

Préparer le Serveur sur DigitalOcean

	1.	Créer un Droplet :
	•	Connecte-toi à ton compte DigitalOcean et crée un nouveau Droplet.
	•	Choisis une image Ubuntu et sélectionne les ressources nécessaires.
	•	Configure les paramètres de sécurité (pare-feu, clés SSH, etc.).
	2.	Configurer le Serveur :
	•	Connecte-toi à ton Droplet via SSH.
	•	Mets à jour le système :

sudo apt update && sudo apt upgrade -y


	•	Installe Python, pip et d’autres dépendances si ce n’est pas déjà fait.

Déployer le Backend Flask

	1.	Cloner le Projet sur le Serveur :

git clone https://github.com/ton-repo/shopify-image-generator.git
cd shopify-image-generator


	2.	Configurer l’Environnement Virtuel :

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt


	3.	Configurer les Variables d’Environnement :
	•	Crée un fichier .env sur le serveur avec les mêmes variables que localement.

nano .env

	•	Ajoute les lignes suivantes :

STABILITY_API_KEY=ta_clé_api_stability_ai
SHOPIFY_API_KEY=ta_clé_api_shopify
SHOPIFY_API_SECRET=ton_secret_api_shopify
REDIRECT_URI=https://pocheetfils.digitalocean.app/auth/callback


	4.	Configurer Gunicorn et Nginx :
	•	Installer Gunicorn (déjà dans requirements.txt).
	•	Installer Nginx :

sudo apt install nginx


	•	Configurer Nginx :

sudo nano /etc/nginx/sites-available/shopify-image-generator

	•	Ajoute la configuration suivante :

server {
    listen 80;
    server_name pocheetfils.digitalocean.app;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /path/to/shopify-image-generator/static/;
    }
}

	•	Remplace /path/to/shopify-image-generator/static/ par le chemin réel de ton dossier static.

	•	Activer la Configuration et Redémarrer Nginx :

sudo ln -s /etc/nginx/sites-available/shopify-image-generator /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx


	5.	Lancer l’Application avec Gunicorn :

gunicorn app:app --bind 127.0.0.1:5000

	•	Pour un déploiement en production, envisage d’utiliser un gestionnaire de processus comme Supervisor ou systemd pour gérer Gunicorn.

Déployer le Frontend React

	1.	Construire l’Application React :

npm run build


	2.	Transférer les Fichiers de Build sur le Serveur :
	•	Utilise scp ou rsync pour copier le dossier build sur le serveur DigitalOcean.

scp -r build/ user@pocheetfils.digitalocean.app:/path/to/shopify-app-frontend/


	3.	Configurer un Serveur Web pour Servir le Frontend :
	•	Tu peux utiliser Nginx pour servir les fichiers statiques React.
	•	Modifier la Configuration Nginx :

sudo nano /etc/nginx/sites-available/shopify-image-generator

	•	Ajoute une location pour le frontend :

server {
    listen 80;
    server_name pocheetfils.digitalocean.app;

    location /api/ {
        proxy_pass http://127.0.0.1:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        root /path/to/shopify-app-frontend/build;
        try_files $uri /index.html;
    }

    location /static/ {
        alias /path/to/shopify-image-generator/static/;
    }
}

	•	Remplace /path/to/shopify-app-frontend/build par le chemin réel de ton dossier build.

	•	Redémarrer Nginx :

sudo systemctl restart nginx

Tester et Finaliser l’App

Tests Locaux

	1.	Exécuter le Backend :

python app.py


	2.	Exécuter le Frontend :

npm start


	3.	Tester les Fonctionnalités :
	•	Ouvre l’application React dans ton navigateur.
	•	Entre un prompt et génère des images.
	•	Vérifie que les images sont traitées et affichées correctement.

Tests en Production

	1.	Accéder à l’App Déployée :
	•	Navigue vers l’URL de ton frontend déployé sur DigitalOcean.
	•	Effectue les mêmes tests qu’en local pour assurer le bon fonctionnement en production.

Soumettre à l’App Store Shopify

	1.	Préparer la Fiche de l’App :
	•	Fournis une description claire, des captures d’écran et les informations nécessaires sur ton application.
	2.	Soumettre pour Révision :
	•	Dans le tableau de bord Shopify Partner, soumets ton application pour révision.
	•	Réponds aux éventuels retours de Shopify durant le processus de révision.



