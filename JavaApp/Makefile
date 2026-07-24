# Prerequisite variables
SOURCEDIR := src
OUTPUTDIR := build

SOURCES := $(shell find $(SOURCEDIR) -type f -name '*.java')
CLASSES := $(SOURCES:$(SOURCEDIR)/%.java=$(OUTPUTDIR)/%.class)

TARGETS := $(shell find $(SOURCEDIR) -mindepth 1 -maxdepth 1 -type d)
TARGETS := $(TARGETS:$(SOURCEDIR)/%=$(OUTPUTDIR)/%.jar)

# Add all jars to classpath and allow compiling with private API
JCFLAGS := -cp "$(subst $() $(),:,$(wildcard $(SOURCEDIR)/*/):$(wildcard libs/*/*.jar))" -XDignore.symbol.file

all: $(TARGETS) clean_classes
	
# Sometimes the CI produces empty class file resulting in broken jar, so we check and abort here
check_empty_classes:
	@set -e
	@echo "Checking for empty class files"
	@find $(OUTPUTDIR) -path '*/classes/*' -prune -o -name '*.class' -size 0 -print > $(OUTPUTDIR)/empty_classes.txt 2>/dev/null || true
	@if [ -s $(OUTPUTDIR)/empty_classes.txt ]; then \
		echo "Error: found empty class files:"; \
		cat $(OUTPUTDIR)/empty_classes.txt; \
		rm -f $(OUTPUTDIR)/empty_classes.txt; \
		exit 1; \
	fi
	@rm -f $(OUTPUTDIR)/empty_classes.txt

# Rules for building java class
$(CLASSES): $(OUTPUTDIR)/%.class: $(SOURCEDIR)/%.java
	@set -e
	$(eval TARGET=$(word 2, $(subst /, ,$@)))
	$(eval SAFE_CLASSNAME=$(subst $$,$$$$,$(@:$(OUTPUTDIR)/$(TARGET)/%.class=%)))
	@mkdir -p $(OUTPUTDIR)/classes
	@mkdir -p $(dir $@)
	@echo "Compiling $<"
	@$(BOOTJDK)/javac $(JCFLAGS) -d $(OUTPUTDIR)/classes $<
	@test -s "$(OUTPUTDIR)/classes/$(SAFE_CLASSNAME).class" || { echo "Error: javac produced 0-byte $(SAFE_CLASSNAME).class. This usually means required types on the classpath (e.g. LWJGL jars in libs/lwjgl/) are missing or incompatible with this JDK."; exit 1; }
	@cp $(OUTPUTDIR)/classes/$(SAFE_CLASSNAME)*.class $(dir $@)/

$(OUTPUTDIR)/launcher.jar: $(CLASSES) check_empty_classes
	@set -e
	@echo "Creating $@"
	@$(BOOTJDK)/jar -cf $@ -C $(basename $@) .

$(OUTPUTDIR)/lwjgl.jar: $(CLASSES) check_empty_classes
	@set -e
	@echo "Creating $@"
	@mkdir -p $(OUTPUTDIR)/lwjgl_lib
	@cd $(OUTPUTDIR)/lwjgl_lib && find ../../libs/lwjgl -name 'lwjgl*.jar' -exec $(BOOTJDK)/jar -xf {} \; && rm -rf META-INF
	@cp -R $(OUTPUTDIR)/lwjgl/* $(OUTPUTDIR)/lwjgl_lib/
	@$(BOOTJDK)/jar -cf $@ -C $(OUTPUTDIR)/lwjgl_lib .

$(OUTPUTDIR)/patchjna_agent.jar: $(CLASSES)
	@set -e
	@mv $(basename $@)/com/sun/jna/Platform.class $(basename $@)/com/sun/jna/Platform.class.patch
	@echo "Creating $@"
	@$(BOOTJDK)/jar -cfm $@ $(SOURCEDIR)/patchjna_agent.txt -C $(basename $@) .

clean:
	rm -rf $(OUTPUTDIR)

clean_classes:
	rm -rf $(OUTPUTDIR)/classes

.SUFFIXES: .java
.PHONY: all clean clean_classes
