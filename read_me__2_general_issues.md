# Issues with `pip install`-command

### Case 1: Token expired

The error message

    ```bash
    Looking in indexes: https://aws:****@ml-secure-env-884128492600.d.codeartifact.eu-central-1.amazonaws.com/pypi/ml-secure-env-pypi/simple/
    WARNING: 401 Error, Credentials not correct for https://ml-secure-env-884128492600.d.codeartifact.eu-central-1.amazonaws.com/pypi/ml-secure-env-pypi/simple/anndata/
    ERROR: Could not find a version that satisfies the requirement anndata (from versions: none)
    ERROR: No matching distribution found for anndata
    ```

The problem is authentication to CodeArtifact.
pip is using your private index, but it gets 401 (Credentials not correct), so it cannot list packages and then reports “no matching distribution”.
Most likely the CodeArtifact token expired (common after ~12 hours).

The solution:

    ```bash
    aws sts get-caller-identity
    aws codeartifact login --tool pip \
      --domain ml-secure-env \
      --domain-owner 884128492600 \
      --repository ml-secure-env-pypi \
      --region eu-central-1
    ```

---
