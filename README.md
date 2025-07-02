# SysAtt.sh
Atualizador de Sistema Completo - Full System Updater


#!/bin/bash

# Cores
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
CYAN='\033[0;36m'
MAGENTA='\033[0;35m'
NC='\033[0m'

LOGFILE="$HOME/sysatt_lucifer_wsl.log"
touch "$LOGFILE"
START=$(date +%s)

VERBOSE=false
if [[ "$1" == "-v" || "$1" == "--verbose" ]]; then
  VERBOSE=true
fi

log() {
  echo -e "$1" | tee -a "$LOGFILE"
}

run_cmd() {
  if $VERBOSE; then
    bash -c "$1" | tee -a "$LOGFILE"
  else
    bash -c "$1" >> "$LOGFILE" 2>&1
  fi
}

check_root() {
  if [ "$EUID" -ne 0 ]; then
    log "${RED}❌ Execute como root usando: sudo $0${NC}"
    exit 1
  fi
}

detect_environment() {
  if grep -qi microsoft /proc/version; then
    if [ -d /run/WSL ]; then
      ENVIRONMENT="WSL2"
    else
      ENVIRONMENT="WSL1"
    fi
  else
    ENVIRONMENT="Linux"
  fi
  log "${CYAN}🧠 Ambiente detectado: $ENVIRONMENT${NC}"
}

backup_wsl() {
  if [[ "$ENVIRONMENT" == "WSL1" || "$ENVIRONMENT" == "WSL2" ]]; then
    DISTRO=$(wsl -l --quiet | head -n 1)
    BACKUP_FILE="$HOME/${DISTRO}_backup_$(date +%F).tar"
    log "${YELLOW}💾 Criando backup da distro WSL ($DISTRO)...${NC}"
    powershell.exe "wsl --export $DISTRO \"$BACKUP_FILE\"" > /dev/null 2>&1
    log "${GREEN}✔️ Backup salvo em: ${BACKUP_FILE}${NC}"
  fi
}

check_kali_gpg() {
  log "${YELLOW}> Verificando chave GPG do repositório Kali...${NC}"
  if ! gpg --list-keys | grep -q "ED65462EC8D5E4C5"; then
    log "${CYAN}> Baixando nova chave GPG...${NC}"
    run_cmd "wget https://archive.kali.org/archive-keyring.gpg -O /usr/share/keyrings/kali-archive-keyring.gpg"
  else
    log "${GREEN}✔️ Chave GPG já instalada.${NC}"
  fi
}

check_connectivity() {
  log "${CYAN}> Verificando conectividade...${NC}"
  if ! ping -c 1 http.kali.org &>/dev/null; then
    log "${RED}❌ Sem conexão com os repositórios. Verifique sua rede.${NC}"
    exit 1
  fi
}

fix_sources_kali() {
  log "${YELLOW}⚠️ Ajustando sources.list para mirror oficial...${NC}"
  cp /etc/apt/sources.list /etc/apt/sources.list.bak
  cat > /etc/apt/sources.list <<EOF
deb http://http.kali.org/kali kali-rolling main non-free contrib
deb-src http://http.kali.org/kali kali-rolling main non-free contrib
EOF
  run_cmd "apt-get clean"
  run_cmd "apt-get update --fix-missing"
}

fix_dependencies() {
  run_cmd "apt --fix-broken install -y"
  run_cmd "dpkg --configure -a"
  run_cmd "apt-get install -f -y"
}

check_and_install_packages() {
  REQUIRED_PKGS=("gettext" "python3-setuptools" "autoconf")
  for pkg in "${REQUIRED_PKGS[@]}"; do
    if ! dpkg -s "$pkg" &>/dev/null; then
      log "${YELLOW}> Instalando $pkg...${NC}"
      run_cmd "apt-get install -y $pkg"
    fi
  done
}

update_and_upgrade() {
  log "${GREEN}> Atualizando pacotes...${NC}"
  run_cmd "apt-get update"
  run_cmd "apt-get upgrade -y | tee /tmp/upgrade_sysatt.log"
  run_cmd "apt-get dist-upgrade -y | tee /tmp/distupgrade_sysatt.log"
  apt list --upgradable 2>/dev/null > /tmp/pacotes_nao_atualizados.txt
}

clean_up() {
  log "${CYAN}> Limpando pacotes desnecessários...${NC}"
  run_cmd "apt autoremove -y"
}

check_reboot_required() {
  if [ -f /var/run/reboot-required ]; then
    log "${RED}⚠️ REBOOT NECESSÁRIO${NC}"
  else
    log "${GREEN}✔️ Nenhum reboot necessário.${NC}"
  fi
}

generate_terminal_report() {
  log ""
  log "${MAGENTA}================= RELATÓRIO =================${NC}"
  log "${CYAN}✔️ VERSÃO DO SISTEMA:${NC}"
  lsb_release -a 2>/dev/null

  log ""
  log "${CYAN}✔️ PACOTES ATUALIZADOS:${NC}"
  grep "^Instalando " /tmp/upgrade_sysatt.log /tmp/distupgrade_sysatt.log 2>/dev/null | awk '{print $2}' | sort -u || echo "Nenhum pacote identificado."

  log ""
  log "${CYAN}❌ PACOTES NÃO ATUALIZADOS:${NC}"
  if grep -q "upgradable" /tmp/pacotes_nao_atualizados.txt 2>/dev/null; then
    grep -v "Listing" /tmp/pacotes_nao_atualizados.txt
  else
    log "${GREEN}Todos os pacotes elegíveis foram atualizados.${NC}"
  fi

  END=$(date +%s)
  DURATION=$(( END - START ))
  log ""
  log "${GREEN}⏱️ Tempo total: $((DURATION / 60)) min $((DURATION % 60)) s${NC}"
  check_reboot_required
  log "${MAGENTA}=============================================${NC}"
}

show_menu() {
  echo -e "${MAGENTA}"
  echo "╔════════════════════════════════════╗"
  echo "║        SysAtt - Atualizador        ║"
  echo "╠════════════════════════════════════╣"
  echo "║ [1] Atualização completa           ║"
  echo "║ [2] Verificar dependências         ║"
  echo "║ [3] Backup + atualização           ║"
  echo "║ [0] Sair                           ║"
  echo "╚════════════════════════════════════╝"
  echo -e "${NC}"
  read -p "Opção: " OPT
}

banner() {
  log ""
  log "${MAGENTA}=====================================${NC}"
  log "${CYAN}         Criado por ${RED}Lucifer         ${NC}"
  log "${MAGENTA}=====================================${NC}"
  log ""
}

# Execução principal
check_root
detect_environment
banner
show_menu

case "$OPT" in
  1)
    check_kali_gpg
    check_connectivity
    fix_sources_kali
    fix_dependencies
    check_and_install_packages
    update_and_upgrade
    clean_up
    fix_dependencies
    generate_terminal_report
    ;;
  2)
    fix_dependencies
    check_and_install_packages
    log ""
    log "${GREEN}✔️ Verificação concluída: Nenhuma dependência crítica pendente.${NC}"
    log "${GREEN}✔️ Tudo certo com o sistema!${NC}"
    log ""
    ;;
  3)
    backup_wsl
    check_kali_gpg
    check_connectivity
    fix_sources_kali
    fix_dependencies
    check_and_install_packages
    update_and_upgrade
    clean_up
    generate_terminal_report
    ;;
  0)
    echo "Saindo..."
    exit 0
    ;;
  *)
    echo "Opção inválida."
    ;;
esac
