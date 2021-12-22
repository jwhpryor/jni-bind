load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Bazel Rules.
http_archive(
  name = "rules_cc",
  urls = ["https://github.com/bazelbuild/rules_cc/archive/40548a2974f1aea06215272d9c2b47a14a24e556.zip"],
  strip_prefix = "rules_cc-40548a2974f1aea06215272d9c2b47a14a24e556",
)

# GUnit.
http_archive(
  name = "googletest",
  urls = ["https://github.com/google/googletest/archive/011959aafddcd30611003de96cfd8d7a7685c700.zip"],
  strip_prefix = "googletest-011959aafddcd30611003de96cfd8d7a7685c700",
)

# Rules Jvm.
RULES_JVM_EXTERNAL_TAG = "4.2"
RULES_JVM_EXTERNAL_SHA = "cd1a77b7b02e8e008439ca76fd34f5b07aecb8c752961f9640dea15e9e5ba1ca"

http_archive(
    name = "rules_jvm_external",
    strip_prefix = "rules_jvm_external-%s" % RULES_JVM_EXTERNAL_TAG,
    sha256 = RULES_JVM_EXTERNAL_SHA,
    url = "https://github.com/bazelbuild/rules_jvm_external/archive/%s.zip" % RULES_JVM_EXTERNAL_TAG,
)

# Google Truth.
load("@rules_jvm_external//:defs.bzl", "maven_install")

maven_install(
    artifacts = [
        "com.google.truth:truth:1.1",
    ],
    repositories = [
        # Private repositories are supported through HTTP Basic auth
        "https://maven.google.com",
        "https://repo1.maven.org/maven2",
    ],
)
