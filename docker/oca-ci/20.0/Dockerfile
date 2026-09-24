FROM ghcr.io/oca/oca-ci/py3.12-odoo20.0:latest

# Core de Odoo refrescado aca, y no el que trae la base. OCA no publica esta
# imagen desde el 22-sep-2026: su CI la prueba contra postgres:12, y Odoo 20.0
# ya usa SQL de PG16 (`any_value`, desde 1316a245c09), asi que los tests fallan y
# el :latest queda congelado. Con eso el community de los builds se atrasa
# respecto del espejo bmya/enterprise@20.0, que es justo el invariante que
# este repo cuida (ver el comentario del workflow): el primer sintoma fue la
# demo de `account` en es_419 (codigo de diario > 7, corregido upstream en
# 347719b4e1b). Nuestro runboat corre PG16, asi que el core nuevo anda.
#
# Misma receta que el Dockerfile de OCA: el ADD del ref invalida la capa cuando
# upstream avanza, y el install editable queda apuntando a /opt/odoo. Las
# dependencias las sigue poniendo la base (requirements de OCB); si Odoo
# agrega una en setup.py, el `pip install -e` la instala.
#
# Cuando OCA vuelva a publicar (`/tmp/odoo-version.json` de su :latest mas
# nuevo que el 22-sep), esto sobra y se puede sacar.
ADD https://api.github.com/repos/odoo/odoo/git/refs/heads/20.0 /tmp/odoo-version.json
RUN rm -rf /opt/odoo \
    && mkdir /tmp/getodoo \
    && (curl -sSL https://github.com/odoo/odoo/tarball/20.0 | tar -C /tmp/getodoo -xz) \
    && mv /tmp/getodoo/* /opt/odoo \
    && rmdir /tmp/getodoo \
    && pip install --no-cache-dir -e /opt/odoo --config-setting=editable_mode=compat \
    && python3 -c "\
import json, odoo.release as r; \
sha = json.load(open('/tmp/odoo-version.json'))['object']['sha']; \
assert r.serie == '20.0', 'serie inesperada: ' + r.serie; \
print('core de Odoo ' + r.serie + ' refrescado a ' + sha[:12])"

# Dependencias opcionales de Odoo 20 que la base no trae. Sin ellas los builds
# arrancan igual, pero degradados y con warnings en el log:
# - libmagic1: python-magic esta en el requirements.txt de Odoo 20 y la base lo
#   instala, pero sin la libreria del sistema no carga ("failed to find
#   libmagic") y odoo.tools.mimetypes cae a su detector propio, menos preciso.
# - markdown2: no esta en el requirements.txt de Odoo. La usan `mail` (vista
#   previa de adjuntos .md del chatter), `ai` de enterprise (pasa a HTML las
#   respuestas de los agentes y los campos generados por IA) y `documents`
#   (vista previa de .md). bmya/odoo-bmya#122 la suma al requirements.txt de
#   odoo-bmya, pero runboat no instala ese archivo: oca_install_addons solo lee
#   test-requirements.txt y los pyproject.toml de cada addon.
# OCA/oca-ci#132 agrega markdown2 a la base; cuando se publique, esa parte sobra.
RUN apt-get update -qq \
    && DEBIAN_FRONTEND=noninteractive apt-get install -qq --no-install-recommends libmagic1 \
    && rm -rf /var/lib/apt/lists/* \
    && pip install --no-cache-dir "markdown2==2.5.5" \
    && python3 -c "\
import magic, markdown2; \
assert magic.from_buffer(b'%PDF-1.4\n', mime=True) == 'application/pdf', 'libmagic no detecta un PDF'; \
html = markdown2.markdown('## T\n\n**b**', extras=['fenced-code-blocks', 'tables', 'strike', 'cuddled-lists']); \
assert '<h2>T</h2>' in html and '<strong>b</strong>' in html, 'markdown2 no renderiza: ' + html; \
print('libmagic OK, markdown2 ' + markdown2.__version__ + ' OK')"

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
