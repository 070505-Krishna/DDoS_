# 07. Implementation Guide (Runnable Code)

All code is PyTorch, short, with simple variable names. Test each block on a small dataset first. Library APIs change between versions, so check official docs for Flower/Opacus.

## 1. Data: synthetic stand-in for a tabular IDS dataset

Replace this with your preprocessed CIC-IDS / UNSW-NB15 / N-BaIoT arrays.

```python
import numpy as np, torch, torch.nn as nn, copy
from torch.utils.data import TensorDataset, DataLoader
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

X, y = make_classification(n_samples=20000, n_features=40, n_informative=20,
                           n_classes=4, n_clusters_per_class=1, random_state=0)
X = StandardScaler().fit_transform(X).astype("float32")
Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=0.2, random_state=0, stratify=y)
```

## 2. Non-IID partition with Dirichlet

```python
def dirichlet_split(y, k, alpha, seed=0):
    rng = np.random.default_rng(seed)
    idx = [[] for _ in range(k)]
    for c in np.unique(y):
        ids = np.where(y == c)[0]; rng.shuffle(ids)
        p = rng.dirichlet([alpha] * k)
        cuts = (np.cumsum(p) * len(ids)).astype(int)[:-1]
        for i, part in enumerate(np.split(ids, cuts)):
            idx[i] += part.tolist()
    return idx

K = 10
parts = dirichlet_split(ytr, K, alpha=0.3)   # smaller alpha = more non-IID

def loader(ids, bs=64):
    ds = TensorDataset(torch.tensor(Xtr[ids]), torch.tensor(ytr[ids]))
    return DataLoader(ds, batch_size=bs, shuffle=True)

loaders = [loader(p) for p in parts]
sizes = [len(p) for p in parts]
test_dl = DataLoader(TensorDataset(torch.tensor(Xte), torch.tensor(yte)), batch_size=512)
```

## 3. Model and helpers

```python
def make_model():
    return nn.Sequential(nn.Linear(40, 64), nn.ReLU(), nn.Linear(64, 32), nn.ReLU(), nn.Linear(32, 4))

def evaluate(m):
    m.eval(); ok = 0; n = 0
    with torch.no_grad():
        for x, t in test_dl:
            ok += (m(x).argmax(1) == t).sum().item(); n += len(t)
    return ok / n
```

## 4. FedAvg from scratch (shortest version)

```python
def local_train(gm, dl, ep=2, lr=0.05):
    m = copy.deepcopy(gm); opt = torch.optim.SGD(m.parameters(), lr=lr); m.train()
    for _ in range(ep):
        for x, t in dl:
            opt.zero_grad(); nn.functional.cross_entropy(m(x), t).backward(); opt.step()
    return m.state_dict()

def fedavg(states, ns):
    tot = sum(ns); avg = copy.deepcopy(states[0])
    for k in avg:
        avg[k] = sum(s[k].float() * n / tot for s, n in zip(states, ns))
    return avg

gm = make_model()
for r in range(30):
    ids = np.random.choice(K, 5, replace=False)          # C = 0.5
    sts = [local_train(gm, loaders[i]) for i in ids]
    gm.load_state_dict(fedavg(sts, [sizes[i] for i in ids]))
    if r % 5 == 0: print(r, round(evaluate(gm), 4))
```

Baselines to run for comparison:
```python
# Centralized: train one model on all Xtr, ytr with same total epochs
# Local-only: train one model per client, average their test accuracy
```

## 5. FedProx (change only the local step)

```python
def local_train_prox(gm, dl, ep=2, lr=0.05, mu=0.01):
    m = copy.deepcopy(gm); opt = torch.optim.SGD(m.parameters(), lr=lr)
    g = [p.detach().clone() for p in gm.parameters()]
    for _ in range(ep):
        for x, t in dl:
            opt.zero_grad()
            loss = nn.functional.cross_entropy(m(x), t)
            loss += (mu / 2) * sum(((p - q) ** 2).sum() for p, q in zip(m.parameters(), g))
            loss.backward(); opt.step()
    return m.state_dict()
```

## 6. Server-side optimizer (FedAvgM)

```python
vel = None; beta = 0.9
def server_step(gm, avg_state, lr=1.0):
    global vel
    gs = gm.state_dict(); new = {}
    if vel is None: vel = {k: torch.zeros_like(v, dtype=torch.float) for k, v in gs.items()}
    for k in gs:
        d = gs[k].float() - avg_state[k].float()          # pseudo-gradient
        vel[k] = beta * vel[k] + d
        new[k] = gs[k].float() - lr * vel[k]
    gm.load_state_dict(new)
```

## 7. Flower simulation (classic NumPyClient style)

Flower versions differ. This shows the classic API (`flwr` 1.x with `start_simulation`); newer versions use ClientApp/ServerApp with `flwr run`. Check the docs for your installed version.

```python
import flwr as fl
from collections import OrderedDict

def get_w(m): return [v.cpu().numpy() for v in m.state_dict().values()]
def set_w(m, w): m.load_state_dict(OrderedDict({k: torch.tensor(v) for k, v in zip(m.state_dict().keys(), w)}))

class Client(fl.client.NumPyClient):
    def __init__(self, i): self.i = i; self.m = make_model()
    def get_parameters(self, config): return get_w(self.m)
    def fit(self, w, config):
        set_w(self.m, w); st = local_train(self.m, loaders[self.i])
        self.m.load_state_dict(st); return get_w(self.m), sizes[self.i], {}
    def evaluate(self, w, config):
        set_w(self.m, w); return 0.0, len(Xte), {"acc": evaluate(self.m)}

fl.simulation.start_simulation(
    client_fn=lambda cid: Client(int(cid)).to_client(),
    num_clients=K,
    config=fl.server.ServerConfig(num_rounds=20),
    strategy=fl.server.strategy.FedAvg(fraction_fit=0.5, min_available_clients=K),
)
```

To use a different algorithm, swap the strategy (`FedProx`, `FedAdam`, `FedMedian`, `Krum`, or a custom subclass of `FedAvg`).

## 8. Attacks

### 8.1 Label flipping (data poisoning)
```python
def flip_labels(y, src, dst):
    y = y.copy(); y[y == src] = dst; return y

bad = [0, 1]                                   # malicious client ids
for i in bad:
    yb = flip_labels(ytr[parts[i]], 1, 0)      # attack class 1 -> labeled as 0 ("benign")
    ds = TensorDataset(torch.tensor(Xtr[parts[i]]), torch.tensor(yb))
    loaders[i] = DataLoader(ds, batch_size=64, shuffle=True)
```

### 8.2 Model poisoning: scaling and sign flip
```python
def poison_state(st, gstate, scale=5.0, flip=False):
    out = {}
    for k in st:
        d = st[k].float() - gstate[k].float()
        out[k] = gstate[k].float() + (-scale if flip else scale) * d
    return out
```

### 8.3 Backdoor (trigger on a few features)
```python
def add_trigger(x, val=3.0):
    x = x.copy(); x[:, :3] = val; return x    # fixed pattern in first 3 features

def poison_client(i, target=0, frac=0.3):
    ids = np.array(parts[i]); n = int(frac * len(ids))
    pick = np.random.choice(len(ids), n, replace=False)
    xb = Xtr[ids].copy(); yb = ytr[ids].copy()
    xb[pick] = add_trigger(xb[pick]); yb[pick] = target
    ds = TensorDataset(torch.tensor(xb), torch.tensor(yb))
    loaders[i] = DataLoader(ds, batch_size=64, shuffle=True)

def asr(m, target=0):                          # attack success rate on non-target test samples
    m.eval(); mask = yte != target
    xt = torch.tensor(add_trigger(Xte[mask]))
    with torch.no_grad(): p = m(xt).argmax(1)
    return (p == target).float().mean().item()
```
Report **main-task accuracy and ASR** every round.

## 9. Defenses (robust aggregation)

Flatten helper then median, trimmed mean, Krum, clipping:

```python
def flat(st): return torch.cat([v.float().flatten() for v in st.values()])
def unflat(vec, ref):
    out, i = {}, 0
    for k, v in ref.items():
        n = v.numel(); out[k] = vec[i:i+n].view_as(v).clone(); i += n
    return out

def agg_median(sts):
    M = torch.stack([flat(s) for s in sts]); return unflat(M.median(0).values, sts[0])

def agg_trim(sts, b=0.2):
    M = torch.stack([flat(s) for s in sts]).sort(0).values
    c = int(b * len(sts)); return unflat(M[c:len(sts)-c].mean(0), sts[0])

def agg_krum(sts, f=1, m=1):
    M = torch.stack([flat(s) for s in sts]); n = len(sts)
    D = torch.cdist(M, M) ** 2
    sc = [D[i].sort().values[1:n-f-1].sum() for i in range(n)]   # n-f-2 nearest
    best = torch.tensor(sc).argsort()[:m]
    return unflat(M[best].mean(0), sts[0])

def clip_updates(sts, gstate, c=1.0):
    out = []
    g = flat(gstate)
    for s in sts:
        d = flat(s) - g; d = d * min(1.0, c / (d.norm().item() + 1e-12))
        out.append(unflat(g + d, s))
    return out
```
Use `fedavg`, `agg_median`, `agg_trim` or `agg_krum` as the server rule; combine `clip_updates` before them. Note: BatchNorm integer buffers (`num_batches_tracked`) are cast to float here; if you use BN, aggregate only float parameters and handle buffers separately.

## 10. DP-FedAvg (client-level, central DP)

```python
def dp_fedavg(sts, gstate, clip=1.0, sigma=0.5):
    g = flat(gstate); ds = []
    for s in sts:
        d = flat(s) - g; d = d * min(1.0, clip / (d.norm().item() + 1e-12)); ds.append(d)
    avg = torch.stack(ds).mean(0)
    avg += torch.randn_like(avg) * sigma * clip / len(sts)        # noise on the mean
    return unflat(g + avg, sts[0])
```
Use the **same** client count each round with fixed sampling probability and compute epsilon with an accountant (e.g. RDP). The noise multiplier `sigma` and the number of rounds determine epsilon. Report (epsilon, delta), e.g. delta = 1/n.

Record-level DP inside local training (Opacus):
```python
from opacus import PrivacyEngine
m = make_model(); opt = torch.optim.SGD(m.parameters(), lr=0.05)
pe = PrivacyEngine()
m, opt, dl = pe.make_private(module=m, optimizer=opt, data_loader=loaders[0],
                             noise_multiplier=1.0, max_grad_norm=1.0)
# train as usual with (m, opt, dl); then pe.get_epsilon(delta=1e-5)
```
Opacus does not support BatchNorm; use GroupNorm/LayerNorm or no norm layer.

## 11. Toy secure aggregation (educational)

Shows masks canceling; not a secure implementation (no key agreement, no dropout handling).

```python
def secagg_demo(vecs, seed=0):
    g = torch.Generator().manual_seed(seed); n = len(vecs); masked = [v.clone() for v in vecs]
    for i in range(n):
        for j in range(i + 1, n):
            m = torch.randn(vecs[0].shape, generator=g)   # shared pairwise mask
            masked[i] += m; masked[j] -= m
    return masked          # server sums these -> equals sum of originals

vs = [torch.randn(5) for _ in range(4)]
mk = secagg_demo(vs); print(torch.allclose(sum(vs), sum(mk), atol=1e-5))
```
For real work use a vetted protocol/library (Flower's SecAgg+, TF Federated secure aggregation, or an MPC library).

## 12. Gradient leakage demo (DLG-style, for privacy evaluation)

```python
def dlg(model, g_true, shape, iters=200):
    x = torch.randn(shape, requires_grad=True); y = torch.randn(1, 4, requires_grad=True)
    opt = torch.optim.LBFGS([x, y])
    for _ in range(iters):
        def closure():
            opt.zero_grad()
            loss = torch.sum(-torch.softmax(y, -1) * torch.log_softmax(model(x), -1))
            g = torch.autograd.grad(loss, model.parameters(), create_graph=True)
            d = sum(((a - b) ** 2).sum() for a, b in zip(g, g_true)); d.backward(); return d
        opt.step(closure)
    return x.detach()
# g_true = gradients from one real sample; compare returned x with the real sample (MSE).
```
Use only on your own models for evaluating the privacy risk. Works best with batch size 1, no noise, smooth activations (use `Sigmoid`/`Tanh` for the demo). Then show that DP noise or larger batches defeat it.

## 13. Membership inference (simple loss-threshold baseline)

```python
def mia_scores(m, x, y):
    m.eval()
    with torch.no_grad(): return -nn.functional.cross_entropy(m(torch.tensor(x)), torch.tensor(y), reduction="none")
# Compute on members (a client's train data) and non-members (held-out); AUC of sklearn roc_auc_score.
```

## 14. Federated autoencoder for anomaly detection

```python
ae = lambda: nn.Sequential(nn.Linear(40, 16), nn.ReLU(), nn.Linear(16, 8), nn.ReLU(), nn.Linear(8, 16), nn.ReLU(), nn.Linear(16, 40))
def ae_train(gm, X, ep=3, lr=1e-3):
    m = copy.deepcopy(gm); opt = torch.optim.Adam(m.parameters(), lr=lr); X = torch.tensor(X)
    for _ in range(ep):
        for i in range(0, len(X), 64):
            b = X[i:i+64]; opt.zero_grad(); nn.functional.mse_loss(m(b), b).backward(); opt.step()
    return m.state_dict()
# Train only on benign samples per client (class 0 here), aggregate with fedavg.
# Score = per-sample MSE; threshold = mean + 3*std on benign validation; evaluate AUC/F1 on mixed test.
```

## 15. Putting experiments in a loop

```python
def run(rule="avg", attack=None, rounds=30, K=10, mal=2):
    # build loaders, mark first `mal` clients malicious, then for each round:
    # 1) sample clients, 2) local train (poisoned for malicious), 3) optional clip,
    # 4) aggregate with rule in {avg, median, trim, krum, dp}, 5) log acc, ASR, round time.
    ...
```
Sweep: `alpha in {0.1, 0.5, 5}`, `malicious fraction in {0, 10, 20, 30%}`, `rule`, `seed in 0..4`; save a CSV of results and plot.

## 16. Debugging checklist

- Accuracy stuck at chance: check label alignment, learning rate, data scaling, model init consistency across clients (all clients must start from the same global weights).
- FedAvg worse than local-only: too many local epochs, extreme non-IID; reduce E, use FedProx, lower LR.
- NaNs: exploding updates in poisoning experiments, missing clipping, inf in dataset.
- Non-reproducible: seeds, `torch.use_deterministic_algorithms`, DataLoader worker seeds.
- Slow simulation: smaller models, fewer rounds, `num_workers=0`, GPU for CNNs.
