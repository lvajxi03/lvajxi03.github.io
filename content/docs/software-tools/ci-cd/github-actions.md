+++
date = '2025-09-04T22:00:13+02:00'
title = 'GitHub Actions'
menus = 'main'
type = 'page'
+++

# Sample build action

```yaml
name: Build

on:
  push:
    tags:
      - '[0-9]+.[0-9]+.[0-9]+'
      - '[0-9]+.[0-9]+.[0-9]+.[0-9]+'

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
    - name: Get tag
      id: tag_name
      run: |
        echo ::set-output name=TAG::$(echo $GITHUB_REF | cut -d / -f 3)
    - name: Display tag
      run: |
        echo "Current tag: ${TAG}"
      env:
        TAG: ${{ steps.tag_name.outputs.TAG }}
    - uses: actions/checkout@v4
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v5
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install build and setuptools
      run: |
        python -m pip install build setuptools
    - name: Update version
      run: |
        sed -i "s/0.0.0/${TAG}/g" setup.cfg
        sed -i "s/0.0.0/${TAG}/g" pyproject.toml
      env:
        TAG: ${{ steps.tag_name.outputs.TAG }}
    - name: Build
      run: |
        python -m build
    - name: Archive artifacts
      uses: actions/upload-artifact@v4
      with:
        name: helik-dist
        path: |
          dist/*.tar.gz
          dist/*.whl
    - name: Publish distribution 📦 to PyPI
      if: startsWith(github.ref, 'refs/tags')
      uses: pypa/gh-action-pypi-publish@master
      with:
        password: ${{ secrets.PYPI_API_TOKEN }}
```        