# GPU Node Auto-Join (on-demand)

Run this once on the GPU machine:

```bash
curl -sfL https://get.k3s.io | K3S_URL="https://pi01:6443" K3S_TOKEN="<same token as in group_vars/all.yaml>" sh -s - agent
sudo mkdir -p /etc/systemd/system/k3s-agent.service.d/
sudo cp gpu-override.conf /etc/systemd/system/k3s-agent.service.d/override.conf
sudo systemctl daemon-reload
sudo systemctl enable --now k3s-agent
```

Install NVIDIA container toolkit and device plugin:

```bash
curl -s -L https://nvidia.github.io/nvidia-container-toolkit/install.sh | sudo bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/nvidia-device-plugin.yml
```

Optionally taint the GPU node (so only GPU workloads land on it):

```bash
kubectl taint nodes gpu01 gpu=true:NoSchedule --overwrite
```

Pods needing GPU should request:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```
