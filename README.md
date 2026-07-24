# DNSSEC_LAB

Module in creation to explain the fundamental terms of DNSSEC, how it works, what it solves, and what it does not do.

## Running the Lab in GitHub Codespaces

This repo includes:

- A Jupyter Notebook to analyze DNSSEC Wireshark captures.

- A containerlab topology (root → tld → auth → resolver → client) that simulates a full DNSSEC chain of trust using BIND9.

### Start

1. On the **`container`** branch, click **Code → Codespaces → Create codespace on container**.
2. Wait for the container to build. The lab deploys automatically via `postStartCommand`.
3. For task 1, open the task-1.ipynb and follow setup instructions
4. For task-2, Start the container:

   ```bash
   cd dnssec-lab 
   sudo containerlab deploy -t dnssec-lab.clab.yml
   ```

   You should see 5 running nodes: `root`, `tld`, `auth`, `resolver`, `client`.

### Disclaimer for task 2

If you do this over multiple codespace sessions you will most likely need to redeploy the containers when you start up the codespace again. Changes will be automatically saved as long as the codespace is only stoppped and not deleted. Either destroy the containers before you end the session or run both commands on startup again.

```bash
   cd dnssec-lab
   sudo containerlab destroy -t dnssec-lab.clab.yml
   sudo containerlab deploy -t dnssec-lab.clab.yml
   ```

### Try it out

```bash
docker exec -it clab-dnssec-lab-client delv @10.0.6.1 ns.example.lab +dnssec -a /etc/bind/root-trust-anchor.key
```

Look for the `RRSIG` record in the answer, and the `ad` flag confirming the resolver validated the full chain.

### Shut down

Stop just the lab (keep the codespace running):

```bash
cd dnssec-lab
sudo containerlab destroy -t dnssec-lab.clab.yml
```

Stop the whole codespace (no compute billing, environment persists):
Code → Codespaces tab → "..." next to the codespace → **Stop codespace**.

Delete the codespace entirely (frees storage):
Code → Codespaces tab → "..." → **Delete**.
