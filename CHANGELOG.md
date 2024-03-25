# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Added a timer for each tracer, and the option of 'strict' timing (with a barrier before and after each tracer)

### Changed

- Reformatted the YAML ExtData file, zero diff
- Changed Be and Pb species to use GOCART convection (instead of MOIST), and GOCART approach (instead of GMI) for settling, dry dep and wet removal; these are now the default settings
- Updated the resistance temperature dependence in the GMI DryDep routine

### Fixed

- Update CI to use Baselibs default from the CircleCI orb
- Bug fix for GMI dry deposition, related to solar zenith angle


### Removed
### Deprecated

## [1.1.0] - 2023-06-09

### Changed

- Instead of calling the GMI routine for Solar Zenith Angle, now call a wrapper for the MAPL version
- Update CI to use Baselibs 7.13.0

## [1.0.0] - 2023-01-18

### Added

- Initial release of TR based on TR_GridComp code from GEOSchem_GridComp v1.11.0
