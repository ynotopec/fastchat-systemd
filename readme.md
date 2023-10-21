# Documentation d'installation de FastChat avec HuggingFace

## Prérequis

* Un environnement Linux.
* Accès root pour l'exécution des commandes.

## Installation

```bash
#!/bin/bash

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
mkdir -p ~/python3-venv ~/fastchat
python3 -m venv ~/python3-venv
source ~/python3-venv/bin/activate
python3 -m pip install --no-cache-dir --upgrade fschat
python3 -m pip install --no-cache-dir --upgrade transformers torch accelerate sentencepiece protobuf gradio
python3 -m pip install --no-cache-dir --upgrade bitsandbytes scipy
EOF

# Création des scripts nécessaires
cat <<EOT > /home/ailab/fastchat/env-fastchat.sh
#!/bin/bash
mkdir -p ~/fastchat
cd ~/fastchat
source ~/python3-venv/bin/activate
export modelName=lmsys/vicuna-33b-v1.3
EOT
chmod +x /home/ailab/fastchat/env-fastchat.sh
chown ailab: /home/ailab/fastchat/env-fastchat.sh

cat <<'EOT' > /home/ailab/fastchat/wait-for-message.sh
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
chmod +x /home/ailab/fastchat/wait-for-message.sh
chown ailab: /home/ailab/fastchat/wait-for-message.sh

# Configuration des services systemd
cat <<EOT > /etc/systemd/system/controller.service
[Unit]
Description=fastchat Controller
After=network.target
Requires=network.target
[Service]
ExecStart=/bin/bash -c 'cd /home/ailab/fastchat && source /home/ailab/python3-venv/bin/activate && python3 -m fastchat.serve.controller > /tmp/controller.log 2>&1'
ExecStartPost=/bin/bash -c '/home/ailab/fastchat/wait-for-message.sh /tmp/controller.log "Uvicorn running"'
User=ailab
EnvironmentFile=/home/ailab/fastchat/env-fastchat.sh
[Install]
WantedBy=multi-user.target
EOT

cat <<EOT > /etc/systemd/system/model_worker.service
[Unit]
Description=fastchat Model Worker
After=controller.service
Requires=controller.service
[Service]
WorkingDirectory=/home/ailab/fastchat
Environment="modelName=lmsys/vicuna-33b-v1.3"
ExecStart=/bin/bash -c 'cd /home/ailab/fastchat && source /home/ailab/python3-venv/bin/activate && python3 -m fastchat.serve.model_worker --model-path '"\${modelName}"' $(lspci | grep -iEw "VGA|NVIDIA" >/dev/null 2>&1 ||echo -n "--device cpu" ) > /tmp/model_worker.log 2>&1'
TimeoutStartSec=360
ExecStartPost=/bin/bash -c '/home/ailab/fastchat/wait-for-message.sh /tmp/model_worker.log "Uvicorn running"'
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
ExecStart=/bin/bash -c 'cd /home/ailab/fastchat && source /home/ailab/python3-venv/bin/activate && python3 -m fastchat.serve.gradio_web_server'
User=ailab
EnvironmentFile=/home/ailab/fastchat/env-fastchat.sh
[Install]
WantedBy=multi-user.target
EOT

# Activation et démarrage des services
systemctl daemon-reload
systemctl enable controller model_worker gradio_web_server
pkill -9 -u ailab python3
systemctl restart controller model_worker gradio_web_server

echo "Installation terminée!"
```

L'utilisateur doit simplement exécuter le script avec des droits root pour installer tout ce dont il a besoin. Pour l'utiliser, sauvegardez ce contenu dans un fichier, par exemple `install_fastchat.sh`, rendez-le exécutable avec `chmod +x install_fastchat.sh` et exécutez-le avec `sudo ./install_fastchat.sh`.

## Vérification

Vérifiez que les services sont en cours d'exécution avec:

```bash
systemctl status controller model_worker gradio_web_server
```

## Références

* [Open LLM Leaderboard - HuggingFace](https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard)
* [FastChat GitHub Repository](https://github.com/lm-sys/FastChat#install)

Voilà! Vous avez maintenant configuré et démarré FastChat sur votre système.
