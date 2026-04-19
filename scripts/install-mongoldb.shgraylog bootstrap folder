#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
DEFAULT_LIB_DIR="${SCRIPT_DIR}/../../infra-bash-lib"
LIB_DIR="${INFRA_BASH_LIB_DIR:-${DEFAULT_LIB_DIR}}"

require_library_files() {
	local missing=0

	for file in common.sh health.sh service.sh apt.sh; do
		if [[ ! -f "${LIB_DIR}/${file}" ]]; then
			echo "[FAIL] Missing library file: ${LIB_DIR}/${file}" >&2
			missing=1
		fi
	done

	if (( missing != 0 )); then
		echo "[FAIL] infra-bash-lib not found. Set INFRA_BASH_LIB_DIR or clone the library repo next to this repo." >&2
		exit 1
	fi
}

require_library_files

# shellcheck source=/dev/null
source "${LIB_DIR}/common.sh"
# shellcheck source=/dev/null
source "${LIB_DIR}/health.sh"
# shellcheck source=/dev/null
source "${LIB_DIR}/service.sh"
# shellcheck source=/dev/null
source "${LIB_DIR}/apt.sh"

MONGODB_VERSION="${MONGODB_VERSION:-8.0}"
MONGODB_PACKAGE="mongodb-org"
MONGODB_SERVICE="mongod"
MONGODB_KEYRING="/usr/share/keyrings/mongodb-server-${MONGODB_VERSION}.gpg"
MONGODB_REPO_FILE="/etc/apt/sources.list.d/mongodb-org-${MONGODB_VERSION}.list"

require_root() {
	if [[ "${EUID}" -ne 0 ]]; then
		die "This script must be run as root or with sudo"
	fi
}

get_ubuntu_codename() {
	local codename=""

	if [[ -r /etc/os-release ]]; then
		codename="$(. /etc/os-release && printf '%s' "${VERSION_CODENAME:-}")"
	fi

	if [[ -z "${codename}" ]]; then
		fail "Unable to determine Ubuntu codename"
		return 2
	fi

	printf '%s\n' "${codename}"
	return 0
}

validate_platform() {
	local codename="$1"

	if [[ -z "${codename}" ]]; then
		fail "Usage: validate_platform <codename>"
		return 2
	fi

	if [[ ! -r /etc/os-release ]]; then
		fail "Platform validation failed: /etc/os-release not found"
		return 2
	fi

	. /etc/os-release

	if [[ "${ID:-}" != "ubuntu" ]]; then
		fail "Unsupported operating system: ${ID:-unknown} (Ubuntu required)"
		return 2
	fi

	case "${codename}" in
		noble|jammy|focal)
			pass "Supported Ubuntu release detected: ${codename}"
			return 0
			;;
		*)
			fail "Unsupported Ubuntu codename for MongoDB ${MONGODB_VERSION}: ${codename}"
			return 2
			;;
	esac
}

run_preflight_checks() {
	local disk_status=0
	local mem_status=0
	local cpu_status=0

	step "Running preflight health checks"

	check_disk_free_gb / 45 35 || disk_status=$?
	check_memory_total_gb 10 8 || mem_status=$?
	check_cpu_load 2.0 4.0 || cpu_status=$?

	if (( disk_status == 2 || mem_status == 2 || cpu_status == 2 )); then
		die "Preflight health checks failed"
	fi

	if (( disk_status == 1 || mem_status == 1 || cpu_status == 1 )); then
		warn "Preflight health checks completed with warnings"
	else
		pass "Preflight health checks passed"
	fi
}

install_repo_prerequisites() {
	step "Installing repository prerequisites"

	apt_update || return 2
	apt_install ca-certificates curl gnupg || return 2

	return 0
}

mongodb_repo_configured() {
	local codename="$1"

	if [[ -z "${codename}" ]]; then
		fail "Usage: mongodb_repo_configured <codename>"
		return 2
	fi

	if [[ ! -f "${MONGODB_REPO_FILE}" ]]; then
		warn "MongoDB repo file not found: ${MONGODB_REPO_FILE}"
		return 1
	fi

	if grep -Fq "repo.mongodb.org/apt/ubuntu ${codename}/mongodb-org/${MONGODB_VERSION}" "${MONGODB_REPO_FILE}"; then
		pass "MongoDB repo is configured"
		return 0
	fi

	warn "MongoDB repo file exists but does not match expected configuration"
	return 1
}

configure_mongodb_repo() {
	local codename="$1"

	if [[ -z "${codename}" ]]; then
		fail "Usage: configure_mongodb_repo <codename>"
		return 2
	fi

	step "Configuring MongoDB repository"

	if [[ ! -f "${MONGODB_KEYRING}" ]]; then
		info "Installing MongoDB GPG key"
		curl -fsSL "https://pgp.mongodb.com/server-${MONGODB_VERSION}.asc" \
			| gpg --dearmor -o "${MONGODB_KEYRING}" \
			|| {
				fail "Failed to install MongoDB GPG key"
				return 2
			}
		pass "MongoDB GPG key installed"
	else
		info "MongoDB GPG key already present"
	fi

	cat > "${MONGODB_REPO_FILE}" <<EOF
deb [ signed-by=${MONGODB_KEYRING} ] https://repo.mongodb.org/apt/ubuntu ${codename}/mongodb-org/${MONGODB_VERSION} multiverse
EOF

	pass "MongoDB repo file written: ${MONGODB_REPO_FILE}"
	return 0
}

install_mongodb_package() {
	step "Installing MongoDB package"

	if package_installed "${MONGODB_PACKAGE}"; then
		info "MongoDB package already installed"
		return 0
	fi

	apt_update || return 2
	apt_install "${MONGODB_PACKAGE}" || return 2

	return 0
}

ensure_mongodb_running() {
	step "Ensuring MongoDB service is available"

	service_exists "${MONGODB_SERVICE}" || return 2

	if service_running "${MONGODB_SERVICE}"; then
		info "MongoDB service already running"
	else
		start_service "${MONGODB_SERVICE}" || return 2
	fi

	enable_service "${MONGODB_SERVICE}" || return 2

	return 0
}

main() {
	local ubuntu_codename

	require_root
	run_preflight_checks
	install_repo_prerequisites || die "Failed to install repository prerequisites"

	ubuntu_codename="$(get_ubuntu_codename)" || die "Failed to detect Ubuntu codename"
	validate_platform "${ubuntu_codename}" || die "Unsupported Ubuntu version"

	if ! mongodb_repo_configured "${ubuntu_codename}"; then
		configure_mongodb_repo "${ubuntu_codename}" || die "Failed to configure MongoDB repository"
	fi

	apt_update || die "APT update failed after repository configuration"
	install_mongodb_package || die "MongoDB installation failed"
	ensure_mongodb_running || die "MongoDB service validation failed"

	step "MongoDB validation"
	package_installed "${MONGODB_PACKAGE}" || die "MongoDB package validation failed"
	service_running "${MONGODB_SERVICE}" || die "MongoDB service is not running"

	pass "MongoDB installed and running successfully"
}

main "$@"
