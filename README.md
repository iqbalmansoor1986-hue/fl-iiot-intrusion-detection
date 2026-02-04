# fl-iiot-intrusion-detection
Cross-silo federated learning IDS for IIoT using FedProx with robust aggregation, update clipping/optional DP, non-IID silo simulation, and poisoning robustness evaluation.

# Cross-Silo Federated IDS for IIoT Networks

This repository implements an end-to-end, dataset-driven simulation of a cross-silo federated learning (FL) intrusion detection system (IDS) designed for industrial/IIoT environments. The goal is to enable multiple industrial sites to collaboratively improve detection performance without sharing raw network telemetry.

## Method Summary

Each site is modeled as an independent silo that holds its own local dataset derived from IIoT flow/session telemetry. A lightweight binary IDS classifier is trained under a synchronous FL protocol: a coordinating server broadcasts the current global model, participating sites perform local optimization on their own labeled data, and only model updates are returned for aggregation. The updated global model is redistributed and used for subsequent inference.

The implementation reflects common industrial constraints:
- **Data governance:** raw telemetry never leaves the site; only model updates are shared.
- **Heterogeneity:** sites may have non-identical data distributions (non-IID), emulated via Dirichlet partitioning.
- **Operational feasibility:** local training is lightweight and communication costs are tracked as bytes per round.

## Data Pipeline and Non-IID Silo Simulation

Experiments are offline and dataset-driven. A raw IIoT intrusion dataset (e.g., Edge-IIoTset) is preprocessed into a fixed-dimensional feature matrix:
- irrelevant or high-cardinality columns are removed,
- categorical attributes are one-hot encoded,
- binary labels are derived (benign vs intrusion),
- each silo applies **local normalization** using statistics computed only on its local training split.

To emulate cross-silo behavior, the dataset is partitioned into **K = 10 silos** using a Dirichlet non-IID protocol with selectable concentration parameter. Each silo retains its own train/validation/test split (70/15/15) after partitioning.

## Model and Training Objective

The primary IDS model is a lightweight feed-forward classifier (MLP) suitable for gateway-grade deployment. Training uses a **weighted binary cross-entropy** objective to address class imbalance.

Local training follows the **FedProx** objective, which adds a proximal regularization term to limit drift under heterogeneous client distributions. Each participating silo runs a small number of local epochs per round and returns a model delta relative to the broadcast global parameters.

## Privacy-Oriented Update Protection

Before transmission, each client update is protected to reduce leakage and bound influence:
- **L2 clipping** limits update magnitude,
- **optional Gaussian noise** can be added to align with DP-SGD-style protections.

This mechanism controls update sensitivity and provides a configurable privacy/utility trade-off.

## Aggregation and Robustness

The coordinator supports two aggregation modes:
- **FedAvg** as the baseline,
- **Trimmed Mean** as an integrity-aware aggregation rule when individual client updates are visible, reducing sensitivity to extreme (potentially malicious) updates by trimming per-coordinate outliers.

## Threshold Calibration (Operational IDS Behavior)

Because decision thresholds may vary across sites and operating conditions, the repository includes **per-silo threshold calibration** using validation data. Thresholds are calibrated to target a specified false positive rate (FPR) using benign validation scores, producing site-specific decision thresholds that stabilize alarm behavior under distribution shift.

## Adversarial FL Simulation

To evaluate resilience against FL-stage threats, the simulation includes a simple poisoning model:
- a bounded fraction of participants per round act as malicious clients,
- malicious updates are generated via sign-flip (optionally scaled), emulating model-degradation poisoning.

Robustness is evaluated across grids of malicious participation fraction and poisoning strength, comparing FedAvg and robust aggregation under identical data splits and seeds.

## Outputs and Reproducibility

All experiments run with fixed random seeds and generate per-run artifacts such as final global evaluation metrics, saved model weights, and aggregated sweep tables. The code is designed to support reproducible comparisons across aggregation rules, privacy settings, non-IID levels, and adversarial conditions.
