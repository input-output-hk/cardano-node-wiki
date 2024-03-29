# Building Cardano Node with nix

The [Nix Package Manager][nix] can be installed on most Linux distributions by downloading and
running the installation script:
```
curl -L https://nixos.org/nix/install > install-nix.sh
chmod +x install-nix.sh
./install-nix.sh
```
and following the directions.

Then the [Flake][flake] feature of nix (and IFD support) should be enabled:
```
sudo mkdir -p /etc/nix
cat <<EOF | sudo tee /etc/nix/nix.conf
experimental-features = nix-command flakes
allow-import-from-derivation = true
EOF
```

#### IOHK Binary Cache

To improve build speed, it is possible to set up a binary cache maintained by IOHK (**this is
optional**):
```
sudo mkdir -p /etc/nix
cat <<EOF | sudo tee -a /etc/nix/nix.conf
substituters = https://cache.nixos.org https://cache.iog.io
trusted-public-keys = hydra.iohk.io:f/Ea+s+dFdN+3Y/G+FDgSq+a5NEWhJGzdjvKNGv0/EQ= cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY=
EOF
```

### Building and running with nix

Once Nix is installed, log out and then log back in.
After that you can build the full node package with Mainnet configuration as follows:
```
git clone https://github.com/intersectmbo/cardano-node
cd cardano-node
nix build .#mainnet/node -o mainnet-node-local
./mainnet-node-local/bin/cardano-node-mainnet
```
or run in in one go:
```
nix run github:intersectmbo/cardano-node#mainnet/node
```

If you only want to build just the cardano-node executable, without the configuration bundle:
```
nix build .#cardano-node -o cardano-node-build
```
To build the cardano-cli executable, follow the steps below:
```
nix build .#cardano-cli -o cardano-cli-build
./cardano-cli-build/bin/cardano-cli
```
Or run directly, eg.:
```
nix run .#cardano-cli -- version
```

### Development environments

A shell environment with pre-compiled, cached, cabal dependencies is available with:
```
nix develop
```
An environment with dependencies compiled with profiling enabled is available with:
```
nix develop .#profiled
```

[nix]: https://nixos.org/nix/
[flake]: https://nixos.wiki/wiki/Flakes
