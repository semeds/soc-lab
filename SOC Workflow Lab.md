
## Step 1: Create the resource group

In the Azure portal (or CLI), create a dedicated resource group so everything is easy to find and tear down later:

**Portal:** Resource groups → Create → name it `rg-soc-lab` → pick a region close to you → Review + Create.

**CLI equivalent:**

```bash
az group create --name rg-soc-lab --location eastus
```

## Step 2: Create the Log Analytics workspace

This is where Sentinel stores its data.

**Portal:** Search "Log Analytics workspaces" → Create → Resource group: `rg-soc-lab` → Name: `law-soc-lab` → Region: same as your resource group → Review + Create.

**CLI:**

```bash
az monitor log-analytics workspace create \
  --resource-group rg-soc-lab \
  --workspace-name law-soc-lab \
  --location eastus
```

## Step 3: Enable Microsoft Sentinel on that workspace

**Portal:** Search "Microsoft Sentinel" → + Create → select `law-soc-lab` → Add.

That's it for step 3 — no CLI needed for this part, it's a portal-driven enable action tied to the workspace.

## Step 4: Install the Windows Security Events content

Once Sentinel is enabled: go to **Content management → Content hub**, search "Windows Security Events," and install it if it's not already installed. This gives you the connector you'll use for Azure Arc + AMA later.

---

A couple of things worth doing right now, since you flagged cost control earlier:

- **Set a budget alert** on the subscription (Cost Management + Billing → Budgets) before going further — even a small threshold like $10–20 gives you an early warning.
- Keep everything in `rg-soc-lab` so you can delete the whole resource group in one shot when you're done with a session.

Once this is up, the next step is onboarding your Windows VM to **Azure Arc** (Servers → Add → Add a single server), which is the prerequisite before the AMA/DCR configuration. Want me to walk through that next, or do you want to get these four resources created first and confirm they're showing up correctly?