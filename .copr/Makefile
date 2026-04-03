# COPR Makefile for automated php-ssh2 builds from git master
# Builds PHP extension from latest git commit

# COPR provides outdir, set default for local testing
outdir ?= .

RELEASE := $(shell cat $(CURDIR)/release 2>/dev/null || echo 1)

# Default target for COPR
srpm:
	@echo "=== Starting php-ssh2 SRPM build from GitHub master ==="
	@echo "Output directory: $(outdir)"
	@echo "Working directory: $$(pwd)"
	@echo "User: $$(whoami)"

	# Install git if not available
	@if ! command -v git >/dev/null 2>&1; then \
		echo "=== Git not found, installing it ==="; \
		if command -v dnf >/dev/null 2>&1; then \
			echo "Using DNF to install git..."; \
			dnf install -y git || (echo "ERROR: Failed to install git with dnf" && exit 1); \
		elif command -v yum >/dev/null 2>&1; then \
			echo "Using YUM to install git..."; \
			yum install -y git || (echo "ERROR: Failed to install git with yum" && exit 1); \
		else \
			echo "ERROR: No package manager found to install git"; \
			exit 1; \
		fi; \
		echo "Git installed successfully"; \
	else \
		echo "=== Git is already available ==="; \
	fi

	# Verify git is now available
	@which git || (echo "ERROR: git still not found after installation attempt" && exit 1)

	# Clone pecl-networking-ssh2 from GitHub
	@echo "=== Cloning pecl-networking-ssh2 from GitHub ==="; \
	rm -rf pecl-networking-ssh2-master; \
	git clone --depth=50 https://github.com/php/pecl-networking-ssh2.git pecl-networking-ssh2-master || exit 1; \
	\
	echo "=== Getting php-ssh2 version info ==="; \
	cd pecl-networking-ssh2-master && \
	COMMIT=$$(git rev-parse HEAD); \
	SHORTCOMMIT=$$(git rev-parse --short HEAD); \
	VERSION=$$(sed -n '/#define PHP_SSH2_VERSION/{s/.*"\([^"]*\)".*/\1/p}' php_ssh2.h); \
	if [ -z "$$VERSION" ]; then \
		VERSION=$$(git describe --tags --abbrev=0 2>/dev/null | sed 's/^v\?RELEASE_//;s/_/./g') || VERSION="1.4.1"; \
	fi; \
	echo "Commit: $$COMMIT"; \
	echo "Short commit: $$SHORTCOMMIT"; \
	echo "Version: $$VERSION"; \
	cd ..; \
	\
	echo "=== Determining release number ==="; \
	FULL_VERSION="$$VERSION"_"$$SHORTCOMMIT"; \
	FEDORA_VER=$$(rpm -E %fedora 2>/dev/null || echo "42"); \
	echo "Checking COPR for existing builds of version $$FULL_VERSION on Fedora $$FEDORA_VER"; \
	COPR_URL="https://download.copr.fedorainfracloud.org/results/reversejames/php-ssh2/fedora-$$FEDORA_VER-x86_64"; \
	BUILD_DIRS=$$(curl -s "$$COPR_URL/" 2>/dev/null | grep -oE '[0-9]+-php-ssh2' | head -10); \
	EXISTING_RELEASES=""; \
	for dir in $$BUILD_DIRS; do \
		RPMS=$$(curl -s "$$COPR_URL/$$dir/" 2>/dev/null | grep -oE "php-ssh2-$$VERSION-[0-9]+.*git$$SHORTCOMMIT.*\.fc[0-9]+\.x86_64\.rpm" | head -1); \
		if [ -n "$$RPMS" ]; then \
			REL=$$(echo "$$RPMS" | sed "s/.*-$$VERSION-\([0-9]\+\).*\.fc[0-9]\+\.x86_64\.rpm/\1/"); \
			EXISTING_RELEASES="$$EXISTING_RELEASES $$REL"; \
		fi; \
	done; \
	if [ -n "$$EXISTING_RELEASES" ]; then \
		MAX_REL=$$(echo $$EXISTING_RELEASES | tr ' ' '\n' | sort -n | tail -1); \
		RELEASE_NUM=$$(($$MAX_REL + 1)); \
		echo "Found existing release $$MAX_REL, incrementing to $$RELEASE_NUM"; \
	else \
		RELEASE_NUM=1; \
		echo "No existing builds found for $$FULL_VERSION, starting with release 1"; \
	fi; \
	\
	echo "=== Creating php-ssh2 tarball ==="; \
	tar czf ssh2-$$VERSION.tar.gz -C . pecl-networking-ssh2-master --transform "s/^pecl-networking-ssh2-master/ssh2-$$VERSION/" || exit 1; \
	\
	echo "=== Copying upstream package.xml ==="; \
	cp pecl-networking-ssh2-master/package.xml package.xml || exit 1; \
	\
	echo "=== Generating spec file ==="; \
	( \
	echo '%bcond_without tests'; \
	echo ''; \
	echo '%global pecl_name   ssh2'; \
	echo '%global with_zts    0%{!?_without_zts:%{?__ztsphp:1}}'; \
	echo '%global ini_name    40-%{pecl_name}.ini'; \
	echo '%global sources     %{pecl_name}-%{version}'; \
	echo '%global _configure  ../%{sources}/configure'; \
	echo ''; \
	echo 'Summary:       PHP bindings for libssh2'; \
	echo 'Name:          php-%{pecl_name}'; \
	echo "Version:       $$VERSION"; \
	echo "Release:       $(RELEASE).$${RELEASE_NUM}.git$${SHORTCOMMIT}%{?dist}"; \
	echo 'License:       PHP-3.01'; \
	echo 'URL:           https://github.com/php/pecl-networking-ssh2'; \
	echo 'Source0:       ssh2-%{version}.tar.gz'; \
	echo 'Source1:       package.xml'; \
	echo ''; \
	echo 'BuildRequires: gcc'; \
	echo 'BuildRequires: php-devel >= 7.0'; \
	echo 'BuildRequires: php-pear'; \
	echo 'BuildRequires: libssh2-devel >= 1.2'; \
	echo ''; \
	echo 'Requires:      libssh2%{?_isa} >= 1.2'; \
	echo 'Requires:      php(zend-abi) = %{php_zend_api}'; \
	echo 'Requires:      php(api) = %{php_core_api}'; \
	echo ''; \
	echo 'Provides:      php-pecl-%{pecl_name}          = %{version}'; \
	echo 'Provides:      php-pecl-%{pecl_name}%{?_isa}  = %{version}'; \
	echo 'Provides:      php-pecl(%{pecl_name})         = %{version}'; \
	echo 'Provides:      php-pecl(%{pecl_name})%{?_isa} = %{version}'; \
	echo ''; \
	echo '%description'; \
	echo 'PHP bindings for the libssh2 library providing access to resources'; \
	echo '(shell, remote exec, tunneling, file transfer) on a remote machine'; \
	echo 'using a secure cryptographic transport.'; \
	echo ''; \
	echo 'Package built for PHP %{php_version}'; \
	echo ''; \
	echo '%prep'; \
	echo '%setup -qc'; \
	echo 'cp %{SOURCE1} .'; \
	echo ''; \
	echo '%{?_licensedir:sed -e '"'"'/LICENSE/s/role=\"doc\"/role=\"src\"/'"'"' -i package.xml}'; \
	echo ''; \
	echo 'cd %{sources}'; \
	echo ''; \
	echo '# Sanity check'; \
	echo 'extver=$$(sed -n '"'"'/#define PHP_SSH2_VERSION/{s/.*\"\([^\"]*\)\".*/\1/p}'"'"' php_ssh2.h)'; \
	echo 'if test "x$${extver}" != "x%{version}"; then'; \
	echo '   : Error: Upstream extension version is $${extver}, expecting %{version}.'; \
	echo '   exit 1'; \
	echo 'fi'; \
	echo 'cd ..'; \
	echo ''; \
	echo 'mkdir NTS'; \
	echo '%if %{with_zts}'; \
	echo 'mkdir ZTS'; \
	echo '%endif'; \
	echo ''; \
	echo '# Drop in the bit of configuration'; \
	echo 'cat << '"'"'EOF'"'"' | tee %{ini_name}'; \
	echo '; Enable '"'"'%{summary}'"'"' extension module'; \
	echo 'extension = %{pecl_name}.so'; \
	echo 'EOF'; \
	echo ''; \
	echo '%build'; \
	echo 'cd %{sources}'; \
	echo '%{__phpize}'; \
	echo ''; \
	echo 'cd ../NTS'; \
	echo '%configure \'; \
	echo '    --with-php-config=%{__phpconfig} \'; \
	echo '    --with-ssh2'; \
	echo 'make %{?_smp_mflags}'; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo 'cd ../ZTS'; \
	echo '%configure \'; \
	echo '    --with-php-config=%{__ztsphpconfig} \'; \
	echo '    --with-ssh2'; \
	echo 'make %{?_smp_mflags}'; \
	echo '%endif'; \
	echo ''; \
	echo '%install'; \
	echo '# Install the NTS stuff'; \
	echo 'make -C NTS install INSTALL_ROOT=%{buildroot}'; \
	echo 'install -D -m 644 %{ini_name} %{buildroot}%{php_inidir}/%{ini_name}'; \
	echo ''; \
	echo '# Install XML package description'; \
	echo 'install -D -m 644 package.xml %{buildroot}%{pecl_xmldir}/%{name}.xml'; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo '# Install the ZTS stuff'; \
	echo 'make -C ZTS install INSTALL_ROOT=%{buildroot}'; \
	echo 'install -D -m 644 %{ini_name} %{buildroot}%{php_ztsinidir}/%{ini_name}'; \
	echo '%endif'; \
	echo ''; \
	echo '# Documentation'; \
	echo 'cd %{sources}'; \
	echo 'for i in LICENSE README.md'; \
	echo 'do [ -f $$i ] && install -Dpm 644 $$i %{buildroot}%{pecl_docdir}/%{pecl_name}/$$i'; \
	echo 'done'; \
	echo ''; \
	echo '%check'; \
	echo 'cd %{sources}'; \
	echo ''; \
	echo ': Minimal load test for NTS extension'; \
	echo '%{__php} --no-php-ini \'; \
	echo '    --define extension=%{buildroot}%{php_extdir}/%{pecl_name}.so \'; \
	echo '    --modules | grep '"'"'^%{pecl_name}$$'"'"''; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo ': Minimal load test for ZTS extension'; \
	echo '%{__ztsphp} --no-php-ini \'; \
	echo '    --define extension=%{buildroot}%{php_ztsextdir}/%{pecl_name}.so \'; \
	echo '    --modules | grep '"'"'^%{pecl_name}$$'"'"''; \
	echo '%endif'; \
	echo ''; \
	echo '%files'; \
	echo '%{?_licensedir:%license %{sources}/LICENSE}'; \
	echo '%doc %{pecl_docdir}/%{pecl_name}'; \
	echo '%{pecl_xmldir}/%{name}.xml'; \
	echo ''; \
	echo '%config(noreplace) %{php_inidir}/%{ini_name}'; \
	echo '%{php_extdir}/%{pecl_name}.so'; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo '%config(noreplace) %{php_ztsinidir}/%{ini_name}'; \
	echo '%{php_ztsextdir}/%{pecl_name}.so'; \
	echo '%endif'; \
	echo ''; \
	echo '%changelog'; \
	echo "* $$(date '+%a %b %d %Y') COPR Builder - $$VERSION-$$RELEASE_NUM.git$$SHORTCOMMIT"; \
	echo '- Automated build from GitHub master branch'; \
	echo "- Git commit: $$SHORTCOMMIT"; \
	) > php-ssh2.spec || exit 1; \
	echo "Spec file generated"; \
	\
	echo "=== Building SRPM ==="; \
	rpmbuild -bs \
		--define "_sourcedir $$(pwd)" \
		--define "_specdir $$(pwd)" \
		--define "_builddir $$(pwd)" \
		--define "_srcrpmdir $(outdir)" \
		--define "_rpmdir $$(pwd)" \
		php-ssh2.spec || exit 1; \
	\
	echo "=== SRPM build complete ==="; \
	ls -la $(outdir)/*.src.rpm

clean:
	@rm -f *.tar.gz *.spec *.src.rpm *.xml *.rpm
	@rm -rf pecl-networking-ssh2-master

.PHONY: srpm clean
