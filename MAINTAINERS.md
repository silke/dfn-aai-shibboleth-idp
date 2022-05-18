# Guide for maintaining this repository

## Updating the included collections

Install the collections as specified in the [`requirements.yml`](requirements.yml)-file.

```sh
ansible-galaxy collection install --requirements-file requirements.yml
```

Add the changes to the repository and commit them with an appropriate commit message:

```sh
git add ansible_collections
git commit -m "Update collections"
```

## Testing the roles

Change into each role-directory and run

```sh
molecule test
```

to test the default scenario.
