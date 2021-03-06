# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License

# This Makefile is for building third-party packages from
# tarballs. For autotools-based packages, we configure each of the
# packages to build static PIC binaries which we can safely link into
# a shared libmesos, and build it in-place without installing it (even
# if one runs 'make install' in this directory). Non-autotools based
# packages may be special cases; this Makefile is responsible for
# passing any special make or configure flags that might be required.

BUILT_SOURCES = # Initialized to enable using +=.

# We need to add '--srcdir=.' needed because 'make distcheck' adds
#  '--srcdir=...' when configuring.
CONFIGURE_ARGS = @CONFIGURE_ARGS@ --enable-shared=no --with-pic --srcdir=.

# Directory which holds bundled dependencies
BUNDLED_DIR = $(top_srcdir)/..

include ../../versions.am

BOOST = boost-$(BOOST_VERSION)
GLOG = glog-$(GLOG_VERSION)
GOOGLETEST = googletest-release-$(GOOGLETEST_VERSION)
GMOCK = $(GOOGLETEST)/googlemock
GPERFTOOLS = gperftools-$(GPERFTOOLS_VERSION)
GTEST = $(GOOGLETEST)/googletest
HTTP_PARSER = http-parser-$(HTTP_PARSER_VERSION)
LIBEV = libev-$(LIBEV_VERSION)
PROTOBUF = protobuf-$(PROTOBUF_VERSION)
PICOJSON = picojson-$(PICOJSON_VERSION)

EXTRA_DIST =		\
  $(BOOST).tar.gz	\
  $(GLOG).tar.gz	\
  $(GOOGLETEST).tar.gz	\
  $(GPERFTOOLS).tar.gz	\
  $(HTTP_PARSER).tar.gz	\
  $(LIBEV).tar.gz	\
  $(PROTOBUF).tar.gz	\
  $(PICOJSON).tar.gz

# We need to patch glog in order to deal with a compilation issue when
# compiling with clang (and C++11); see MESOS-860, MESOS-966.
EXTRA_DIST +=		\
  $(GLOG).patch

# We patch http-parser to avoid a name collision with our `flags` namespace.
EXTRA_DIST +=		\
  $(HTTP_PARSER).patch

# We patch libev to keep it from reaping child processes.
EXTRA_DIST +=		\
  $(LIBEV).patch

# We include an upstream patch for protobuf in order to avoid
# signed/unsigned comparisons.
EXTRA_DIST +=		\
  $(PROTOBUF).patch

CLEAN_EXTRACTED =	\
  $(BOOST)		\
  $(GLOG)		\
  $(GOOGLETEST)		\
  $(GPERFTOOLS)		\
  $(HTTP_PARSER)	\
  $(LIBEV)		\
  $(PICOJSON)		\
  $(PROTOBUF)

# This is where the magic happens: we use stamp files as dependencies
# which cause the packages to get extracted as necessary. We also
# apply any patches as appropriate.
%-stamp: $(BUNDLED_DIR)/%.tar.gz
	gzip -d -c $^ | tar xf -
	test ! -e $(top_srcdir)/../$*.patch || patch -d $* -p1 <$(top_srcdir)/../$*.patch
	touch $@

# Dependencies for all-local.
ALL_LOCAL = # Initialize ALL_LOCAL so we can add to it.

if WITH_BUNDLED_BOOST
  ALL_LOCAL += $(BOOST)-stamp
endif

if WITH_BUNDLED_GLOG
LIB_GLOG = $(GLOG)/libglog.la
GLOG_LDFLAGS = # Initialize to empty
if OS_FREEBSD
  GLOG_LDFLAGS += "LDFLAGS='-L/usr/local/lib -lexecinfo'"
endif

$(LIB_GLOG): $(GLOG)-build-stamp

# NOTE: We explicitly set GTEST_CONFIG=no so that we don't attempt to
# build glog against a system installed gtest (there's no reason to
# "test" glog and it's highly likely it will cause a compilation bug
# when building with clang).
$(GLOG)-build-stamp: $(GLOG)-stamp
	cd $(GLOG) && ./configure GTEST_CONFIG=no $(CONFIGURE_ARGS) && \
          $(MAKE) $(AM_MAKEFLAGS) $(GLOG_LDFLAGS)
	touch $@

ALL_LOCAL += $(LIB_GLOG)
endif

if HAS_GPERFTOOLS
LIB_GPERFTOOLS = $(GPERFTOOLS)/libprofiler.la

$(LIB_GPERFTOOLS): $(GPERFTOOLS)-build-stamp

$(GPERFTOOLS)-build-stamp: $(GPERFTOOLS)-stamp
	cd $(GPERFTOOLS) && ./configure  $(CONFIGURE_ARGS) && \
	  $(MAKE) $(AM_MAKEFLAGS)
	touch $@

ALL_LOCAL += $(LIB_GPERFTOOLS)
endif


if WITH_BUNDLED_HTTP_PARSER
# Convenience library for nodejs/http-parser.
noinst_LTLIBRARIES = libry_http_parser.la
nodist_libry_http_parser_la_SOURCES = $(HTTP_PARSER)/http_parser.c
libry_http_parser_la_CPPFLAGS = -I$(HTTP_PARSER) -DHTTP_PARSER_STRICT=0

BUILT_SOURCES += $(nodist_libry_http_parser_la_SOURCES)

$(HTTP_PARSER)/http_parser.c: $(HTTP_PARSER)-stamp
$(HTTP_PARSER)-build-stamp: libry_http_parser.la
	touch $@

ALL_LOCAL += $(HTTP_PARSER)-build-stamp
endif


if WITH_BUNDLED_LIBEV
$(LIBEV)/libev.la: $(LIBEV)-build-stamp
$(LIBEV)-build-stamp: $(LIBEV)-stamp
	cd $(LIBEV) && ./configure $(CONFIGURE_ARGS) && \
          $(MAKE) $(AM_MAKEFLAGS)
	touch $@
ALL_LOCAL += $(LIBEV)-build-stamp
endif

if WITH_BUNDLED_PICOJSON
ALL_LOCAL += $(PICOJSON)-stamp
endif


if WITH_BUNDLED_PROTOBUF
$(PROTOBUF)/src/protoc $(PROTOBUF)/src/libprotobuf.la: $(PROTOBUF)-build-stamp

$(PROTOBUF)-build-stamp: $(PROTOBUF)-stamp
	cd $(PROTOBUF) && ./configure $(CONFIGURE_ARGS) && \
        $(MAKE) $(AM_MAKEFLAGS)
	touch $@
ALL_LOCAL += $(PROTOBUF)/src/libprotobuf.la
ALL_LOCAL += $(PROTOBUF)/src/protoc
endif

# Convenience library for gmock/gtest.
check_LTLIBRARIES = libgmock.la
nodist_libgmock_la_SOURCES = gmock_sources.cc

if WITH_BUNDLED_GMOCK
libgmock_la_CPPFLAGS =			\
  -I$(GTEST)/include -I$(GTEST) -I$(GTEST)/src	\
  -I$(GMOCK)/include -I$(GMOCK) -I$(GMOCK)/src

  gmock_sources.cc: $(GOOGLETEST)-stamp

$(GMOCK)-build-stamp: libgmock.la
	touch $@
ALL_LOCAL += $(GMOCK)-build-stamp
endif

if LINKGTEST
libgmock_la_LIBADD = -lgtest
endif


# We list the sources in BUILT_SOURCES to make sure that the package
# gets unarchived first.
BUILT_SOURCES += $(nodist_libgmock_la_SOURCES)

all-local: $(ALL_LOCAL)

clean-local:
	rm -r -f $(CLEAN_EXTRACTED)
	rm -f *-stamp
