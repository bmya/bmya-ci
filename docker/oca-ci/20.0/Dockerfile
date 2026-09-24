FROM ghcr.io/oca/oca-ci/py3.12-odoo20.0:latest

# Extra Python dependencies required by BMYA modules not present in the base image
RUN pip install --no-cache-dir \
    "pdf417gen==0.7.1" \
    "transbank-sdk==6.1.0" \
    "phonenumbers==9.0.29"

# PyPDF2: a diferencia de 19.0, la base de 20.0 es py3.12 y el requirements.txt
# de Odoo 20 ya elige la 2.12.1 que corre en produccion
# (`PyPDF2==2.12.1 ; python_version < '3.13'`), asi que no se reinstala. Solo se
# verifica: si la base cambiara de Python y nos devolviera una PyPDF2 sin
# `lower_left`, todo test que complete una firma de Enterprise revienta con un
# error que se lee como "el PR rompio sign" (ver docker/oca-ci/19.0/Dockerfile).
RUN python3 -c "\
import PyPDF2; \
from PyPDF2.generic import RectangleObject; \
assert PyPDF2.__version__ == '2.12.1', 'version inesperada: ' + PyPDF2.__version__; \
assert hasattr(RectangleObject, 'lower_left'), 'falta lower_left: sign de Enterprise no va a andar'; \
print('PyPDF2 ' + PyPDF2.__version__ + ' con lower_left OK')"

# pybmyafintoc: el mismo wheel vendorizado que en 19.0, por las mismas razones
# (docker/oca-ci/19.0/Dockerfile): dependencia propietaria de
# account_bank_statement_online_import_fintoc, fuera de PyPI. El wheel es
# py3-none-any y se probo en Python 3.12 (fintoc 2.9.0).
COPY vendor/pybmyafintoc-0.1.6-py3-none-any.whl /tmp/
RUN pip install --no-cache-dir /tmp/pybmyafintoc-0.1.6-py3-none-any.whl \
    && rm /tmp/pybmyafintoc-0.1.6-py3-none-any.whl \
    && python3 -c "\
import importlib.metadata as m; \
from pybmyafintoc.fintoc_client import FintocClient, build_account_index, build_payment_ref, \
    extract_rut, get_counterpart_account, normalize_account_number, normalize_rut, parse_fintoc_date; \
import fintoc.paginator as pag; \
assert m.version('pybmyafintoc') == '0.1.6', 'version inesperada: ' + m.version('pybmyafintoc'); \
assert pag.request.__name__ == '_patched_paginator_request', 'el parche del paginador no se aplico'; \
assert normalize_account_number('0012-3456.789') == '00123456789', 'normalize_account_number cambio de contrato'; \
assert 'REF-1' in build_payment_ref({'description': 'x', 'amount': 1, 'reference_id': 'REF-1'}), 'build_payment_ref cambio de contrato'; \
assert extract_rut('pago de 12.345.678-5') == ['12.345.678-5'], 'extract_rut cambio de contrato'; \
assert parse_fintoc_date('2026-01-15T00:00:00Z').year == 2026, 'parse_fintoc_date cambio de contrato'; \
print('pybmyafintoc ' + m.version('pybmyafintoc') + ' + fintoc ' + m.version('fintoc') + ' OK')"
