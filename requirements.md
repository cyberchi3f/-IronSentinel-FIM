# =============================================================================
#  IronSentinel-FIM  –  Python Dependencies
#  Install:  pip install -r requirements.txt
#
#  For Windows Service support, also run AFTER pip install:
#    python -m pywin32_postinstall -install
# =============================================================================

# Real-time filesystem event monitoring
watchdog>=4.0.0

# Process inspection (PID, exe path, open files, username)
psutil>=5.9.0

# YAML configuration file parsing
PyYAML>=6.0

# Rich terminal output (colours, panels, tables)
rich>=13.0.0

# ANSI colour compatibility shim for older Windows terminals
colorama>=0.4.6

# Windows-native APIs: services, security, registry, ACLs
# Provides: win32api, win32security, win32service, win32serviceutil,
#           servicemanager, win32event, win32con, pywintypes
pywin32>=306

# Optional – faster file event backend (uses ReadDirectoryChangesW directly)
# Only install if watchdog events feel sluggish on very busy paths.
# watchgod>=0.8.3
