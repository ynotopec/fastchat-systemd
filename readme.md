# Documentation d'installation de FastChat avec HuggingFace

## Prérequis

* Un environnement Linux.
* Accès root pour l'exécution des commandes.

## Installation

```bash
#!/bin/bash
#https://github.com/lm-sys/FastChat

# LLM model
#lmsys/vicuna-33b-v1.3
export modelPath=mistralai/Mixtral-8x7B-Instruct-v0.1

export modelName=$(basename "${modelPath}" |sed -rn 's#^(|[0-9]+[bB][^[:alnum:]]+)([[:alnum:]]+)([^[:alnum:]].*|)$#\2#p' |tr '[:upper:]' '[:lower:]' )

# Vérifiez si le script est exécuté en tant que root
if [[ $EUID -ne 0 ]]; then
    echo "Veuillez exécuter ce script en tant que root" 1>&2
    exit 1
fi

# Créer l'utilisateur ailab
userName=ailab
useradd ${userName} -s /bin/bash -d /home/${userName} -g root -m -k /etc/skel

# Mise à jour et installation des dépendances
apt update
apt upgrade -y
apt install python3-venv -y

# Configuration de l'environnement utilisateur
su - ${userName} <<'EOF'
mkdir -p ~/fastchat/.venv
python3 -m venv ~/FastChat/.venv
source ~/FastChat/.venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install --upgrade pip install "fschat[model_worker,webui]"
python3 -m pip install --upgrade pip install fschat transformers torch accelerate sentencepiece protobuf gradio bitsandbytes scipy
EOF

cat <<'EOT' > /home/ailab/FastChat/wait-for-message.sh
#!/bin/bash
LOGFILE="$1"
MESSAGE="$2"
while true; do
  if grep -q "$MESSAGE" "$LOGFILE"; then
    break
  fi
  sleep 1
done
EOT
chmod +x /home/ailab/FastChat/wait-for-message.sh
chown ailab: /home/ailab/FastChat/wait-for-message.sh

# Configuration des services systemd
cat <<EOT > /etc/systemd/system/controller.service
[Unit]
Description=fastchat Controller
After=network.target
Requires=network.target
[Service]
ExecStart=/bin/bash -c 'cd /home/ailab/FastChat && source .venv/bin/activate && python3 -m fastchat.serve.controller > /tmp/controller.log 2>&1'
ExecStartPost=/bin/bash -c '/home/ailab/FastChat/wait-for-message.sh /tmp/controller.log "Uvicorn running"'
User=ailab
[Install]
WantedBy=multi-user.target
EOT

cat <<EOT > /etc/systemd/system/model_worker.service
[Unit]
Description=fastchat Model Worker
After=controller.service
Requires=controller.service
[Service]
WorkingDirectory=/home/ailab/FastChat
Environment="modelPath=${modelPath}"
Environment="modelName=${modelName}"
ExecStart=/bin/bash -c 'cd /home/ailab/FastChat && source .venv/bin/activate && python3 -m fastchat.serve.model_worker --load-8bit --model-names "'"\${modelName}"',gpt-4,gpt-3.5-turbo-instruct,gpt-3.5-turbo,gpt-3.5-turbo-16k,text-davinci-003,text-embedding-ada-002" --model-path '"\${modelPath}"' $(lspci | grep -iEw "VGA|NVIDIA" >/dev/null 2>&1 ||echo -n "--device cpu" ) > /tmp/model_worker.log 2>&1'
TimeoutStartSec=360
ExecStartPost=/bin/bash -c '/home/ailab/FastChat/wait-for-message.sh /tmp/model_worker.log "Uvicorn running"'
User=ailab
[Install]
WantedBy=multi-user.target
EOT

cat <<EOT >/etc/systemd/system/fastchat-openai-api.service
[Unit]
Description=fastchat Api
After=model_worker.service
Requires=model_worker.service

[Service]
ExecStart=/bin/bash -c 'cd /home/ailab/FastChat && source .venv/bin/activate && python3 -m fastchat.serve.openai_api_server --host 0.0.0.0 --port 8501'
User=ailab

[Install]
WantedBy=multi-user.target
EOT

cat <<EOT > /etc/systemd/system/gradio_web_server.service
[Unit]
Description=fastchat Gradio Web Server
After=model_worker.service
Requires=model_worker.service
[Service]
ExecStart=/bin/bash -c 'cd /home/ailab/FastChat && source .venv/bin/activate && python3 -m fastchat.serve.gradio_web_server --port 8502'
User=ailab
[Install]
WantedBy=multi-user.target
EOT

# Activation et démarrage des services
systemctl daemon-reload
systemctl enable controller model_worker gradio_web_server fastchat-openai-api
pkill -9 -u ailab python3
systemctl restart controller model_worker gradio_web_server fastchat-openai-api

echo "Installation terminée!"
```

L'utilisateur doit simplement exécuter le script avec des droits root pour installer tout ce dont il a besoin. Pour l'utiliser, sauvegardez ce contenu dans un fichier, par exemple `install_fastchat.sh`, rendez-le exécutable avec `chmod +x install_fastchat.sh` et exécutez-le avec `sudo ./install_fastchat.sh`.

## Vérification

Vérifiez que les services sont en cours d'exécution avec:

```bash
systemctl status controller model_worker gradio_web_server fastchat-openai-api
```

## Références

* [Open LLM Leaderboard - HuggingFace](https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard)
* [FastChat GitHub Repository](https://github.com/lm-sys/FastChat#install)

Voilà! Vous avez maintenant configuré et démarré FastChat sur votre système.
