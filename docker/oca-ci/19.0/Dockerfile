FROM ghcr.io/oca/oca-ci/py3.10-odoo19.0:latest

# Extra Python dependencies required by BMYA modules not present in the base image
RUN pip install --no-cache-dir \
    "pdf417gen==0.7.1" \
    "transbank-sdk==6.1.0" \
    "phonenumbers==9.0.29"

# PyPDF2 2.12.1: la version que corre en PRODUCCION, y la que el `sign` de
# Enterprise necesita.
#
# La base es py3.10, y el requirements.txt de Odoo 19 elige por version de
# Python:
#
#   PyPDF2==1.26.0 ; python_version <= '3.10'
#   PyPDF2==2.12.1 ; python_version > '3.10' and python_version < '3.13'
#   PyPDF==5.4.0   ; python_version >= '3.13'
#
# O sea que esta imagen se queda con la 1.26.0, mientras las instancias de
# bmya.cloud corren Python 3.12 y por lo tanto la 2.12.1. Esa diferencia no es
# academica: `sign/models/sign_document.py:437` hace `box.lower_left`, un alias
# en snake_case que aparecio recien en PyPDF2 2.x. Con la 1.26 --que solo tiene
# `lowerLeft`-- TODO test que complete una firma revienta con
#
#   AttributeError: 'RectangleObject' object has no attribute 'lower_left'
#
# y el sintoma se lee como "el PR rompio el modulo de firmas", igual que el caso
# del community desactualizado documentado en bmya-infrastructure/CLAUDE.md.
#
# El pin de Odoo es conservador --sigue lo que empaqueta Ubuntu-- y no una
# incompatibilidad: PyPDF2 2.12.1 declara python_requires >=3.6 y anda de sobra
# en 3.10. Lo correcto es que la CI pruebe con la misma libreria que produccion,
# no con la que le tocaria a un Ubuntu mas viejo del que usamos.
#
# El dia que OCA publique una py3.12-odoo19.0, esta linea sale sola: el
# requirements de Odoo ya elegiria la 2.12.1.
RUN pip install --no-cache-dir "PyPDF2==2.12.1" \
    && python3 -c "\
import PyPDF2; \
from PyPDF2.generic import RectangleObject; \
assert PyPDF2.__version__ == '2.12.1', 'version inesperada: ' + PyPDF2.__version__; \
assert hasattr(RectangleObject, 'lower_left'), 'falta lower_left: sign de Enterprise no va a andar'; \
print('PyPDF2 ' + PyPDF2.__version__ + ' con lower_left OK')"

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
