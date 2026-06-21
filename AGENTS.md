# AGENTS.md

## Cursor Cloud specific instructions

Scapy is a single, pure-Python packet-manipulation **library + CLI** (no server,
database, or web frontend). The only hard requirement is a Python 3.7+ interpreter
(this environment uses the system `python3`, 3.12). There are no long-running
services to start.

Dev/test/build dependencies are installed into the user site (`~/.local`) by the
startup update script. Console scripts (`tox`, `flake8`, `mypy`, `codespell`,
`twine`) live in `~/.local/bin`. That directory is added to `PATH` via `~/.bashrc`;
if a non-login shell does not pick it up, invoke tools with `python3 -m <tool>`
instead (e.g. `python3 -m flake8`).

### Running the app
- `./run_scapy` launches the interactive Scapy shell (sets `PYTHONPATH` and runs
  `python3 -m scapy`). Use `PYTHON=python3 ./run_scapy`.
- Sending/receiving real packets opens raw sockets and requires **root**: run
  `sudo PYTHON=python3 ./run_scapy`. Without root, packet crafting/dissection works
  but `sr()/sr1()/send()` raise `PermissionError`.
- Non-interactive scripted run (CI pattern): `./run_scapy -H -c <file.py> </dev/null`
  where the script ends with `sys.exit(0)` — otherwise the shell blocks waiting on a
  TTY and the process aborts.

### Lint / typing
- `python3 -m flake8 scapy/` (config in `tox.ini [flake8]`).
- `tox -e mypy` is the canonical type check. Run mypy through tox (isolated env) —
  running `python3 .config/mypy/mypy_check.py linux` in the main dev env reports
  false `matplotlib`/`Line2D` errors because `matplotlib` is installed there but not
  in the mypy tox env.
- `tox -e spell` runs `codespell`.

### Tests
- Tests use Scapy's own `UTscapy` runner (`.uts` files under `test/`), driven by
  `tox` (envs `py{ver}-linux-{non_root,root}`).
- Non-root run (no sudo): `python3 -m scapy.tools.UTscapy -c ./test/configs/linux.utsc -N -K tshark -K vcan_socket`
  (skip `-K tshark` only if `tshark` is installed; `vcan_socket` needs the `vcan`
  kernel module, unavailable here).
- Root run (full on-the-wire coverage): prefix with `sudo -E`.
- The `Test with DNS over TCP` case makes a **live** DNS-over-TCP request to
  `8.8.8.8:53`; the sandbox resets that connection (`ConnectionResetError`), so this
  single test fails for network reasons only — not a code issue.
- `tcpdump` and `python3-dev` headers (needed to build the optional `brotli`
  compression dep) are already installed in the VM snapshot; do not reinstall them in
  the update script.

### Build
- `SCAPY_VERSION=3.0.0 python3 -m build` produces sdist + wheel; validate with
  `python3 -m twine check --strict dist/*`. (`dist/`, `build/`, `*.egg-info/` are
  gitignored.)
