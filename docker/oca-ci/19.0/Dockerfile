FROM ghcr.io/oca/oca-ci/py3.10-odoo19.0:latest

# Extra Python dependencies required by BMYA modules not present in the base image
RUN pip install --no-cache-dir \
    "pdf417gen==0.7.1" \
    "transbank-sdk==6.1.0" \
    "phonenumbers==9.0.29"

# pybmyafintoc: dependencia propietaria de account_bank_statement_online_import_fintoc
# (bmya-enterprise), declarada en external_dependencies. Odoo valida esa clave con
# importlib.metadata ANTES de importar el módulo, así que sin esto el módulo no
# instala en ningún build que use esta imagen (incluido runboat).
#
# No está en PyPI (repo privado bmya/pybmyafintoc, licencia Proprietary) y este
# repo no tiene configurado un mecanismo de autenticación hacia otros repos
# privados de la organización (deploy key / PAT como build secret), así que se
# vendoriza el wheel acá en vez de instalarlo por git+ssh. Es más feo que un
# índice privado, pero no requiere secretos nuevos y es reproducible sin red.
#
# Pin exacto (no >=): librería de un solo consumidor sin tests publicados; un
# >= metería una versión nueva en producción en el próximo build sin que nadie
# lo pida. Al publicar una versión nueva, actualizar el wheel de vendor/ y este
# pip install en el mismo commit.
#
# httpx y fintoc se resuelven desde PyPI: son dependencias públicas del wheel.
# El rango de fintoc lo fija pybmyafintoc en su pyproject (hoy >=1.0.0,<2.10.0;
# el techo es deliberado, por un choque de cryptography con docsonline). Acá NO
# se repite ese rango a propósito: duplicarlo haría que los dos se desincronicen.
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
