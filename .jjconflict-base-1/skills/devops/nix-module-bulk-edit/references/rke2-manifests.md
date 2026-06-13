# RKE2 manifest injection on NixOS

Use `services.rke2.manifests` for declarative manifest deployment instead of staging files with `environment.etc` + `systemd.tmpfiles.rules`.

Useful option surface from `search.nixos.org` / MyNixOS:
- `services.rke2.manifests.<name>.enable`
- `services.rke2.manifests.<name>.source`
- `services.rke2.manifests.<name>.content`
- `services.rke2.manifests.<name>.target`

Practical pattern:
- `target` should be the filename RKE2 should see under `/var/lib/rancher/rke2/server/manifests`
- `content` can be a Nix attrset; keep it compact and use short-hand syntax when readable
- for HelmChartConfig resources, use a Nix attrset with string values rather than raw YAML strings when possible

Example shape:
```nix
services.rke2.manifests.my-config = {
  target = "my-config.yaml";
  content = {
    apiVersion = "helm.cattle.io/v1";
    kind = "HelmChartConfig";
    metadata = {
      name = "my-chart";
      namespace = "kube-system";
    };
    spec.valuesContent = ''
      foo:
        bar: baz
    '';
  };
};
```

Verification steps after changing RKE2 config:
- `nix-instantiate --parse` or an equivalent Nix syntax check
- `nixos-rebuild switch --flake ...`
- verify the rendered manifest appears under `/var/lib/rancher/rke2/server/manifests`
- verify the resulting HelmChartConfig / addon pods are running
- probe the actual failure mode from a pod if the change is meant to fix networking or DNS
