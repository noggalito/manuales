# 1. Heredar de la imagen oficial
FROM odoo:19.0

# 2. Cambiar a usuario root
USER root

# 3. Instalar dependencias del sistema para librerías de Python complejas
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    python3-dev \
    libldap2-dev \
    libsasl2-dev \
    libsnmp-dev \
    && rm -rf /var/lib/apt/lists/*

# 4. Instalar librerías que suelen pedir los módulos Enterprise 19.0
# Se instalan directamente vía pip ya que no hay un requirements.txt centralizado
RUN pip3 install --no-cache-dir \
    num2words \
    pandas \
    cryptography \
    pyOpenSSL \
    phonenumbers \
    pycountry \
    --break-system-packages

# 5. Volver al usuario odoo
USER odoo