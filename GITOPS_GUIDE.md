# GitOps Management Guide

This guide explains how to manage applications, operators, and infrastructure components on your Kubernetes cluster using ArgoCD and GitOps principles.

## Overview

This cluster uses a GitOps approach where:
- **Ansible** manages cluster bootstrapping (K3s, ArgoCD installation, node configuration)
- **ArgoCD** manages all Kubernetes applications, operators, and infrastructure components
- **Git repositories** serve as the single source of truth for cluster state

## ArgoCD Apps Repository Structure

Your ArgoCD applications should be organized in a Git repository (configured in `argocd_apps_repo_url`) with the following structure:
