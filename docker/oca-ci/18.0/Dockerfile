FROM ghcr.io/oca/oca-ci/py3.10-odoo18.0:latest

# Extra Python dependencies required by BMYA modules not present in the base image
RUN pip install --no-cache-dir \
    "pdf417gen==0.7.1" \
    "pdfminer.six==20231228" \
    "transbank-sdk==6.1.0" \
    "cryptography>=41.0.5,<42" \
    "pyopenssl>=24.3.0,<26"
