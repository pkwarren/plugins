# syntax=docker/dockerfile:1.7
FROM --platform=$BUILDPLATFORM golang:1.22.2-bookworm AS build

ARG TARGETOS TARGETARCH
ENV CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH

WORKDIR /tmp
RUN git clone --depth 1 --branch cmd/protoc-gen-go-grpc/v1.3.0 https://github.com/grpc/grpc-go.git
WORKDIR /tmp/grpc-go
RUN git apply <<'EOF'
diff --git a/cmd/protoc-gen-go-grpc/grpc.go b/cmd/protoc-gen-go-grpc/grpc.go
index 9e15d2d8..753020de 100644
--- a/cmd/protoc-gen-go-grpc/grpc.go
+++ b/cmd/protoc-gen-go-grpc/grpc.go
@@ -20,6 +20,7 @@ package main
 
 import (
 	"fmt"
+	"path"
 	"strconv"
 	"strings"
 
@@ -128,8 +129,27 @@ func generateFile(gen *protogen.Plugin, file *protogen.File) *protogen.Generated
 	if len(file.Services) == 0 {
 		return nil
 	}
-	filename := file.GeneratedFilenamePrefix + "_grpc.pb.go"
-	g := gen.NewGeneratedFile(filename, file.GoImportPath)
+	var g *protogen.GeneratedFile
+	if !*separatePackage {
+		filename := file.GeneratedFilenamePrefix + "_grpc.pb.go"
+		g = gen.NewGeneratedFile(filename, file.GoImportPath)
+	} else {
+		file.GoPackageName += "grpc"
+		dir := path.Dir(file.GeneratedFilenamePrefix)
+		base := path.Base(file.GeneratedFilenamePrefix)
+		file.GeneratedFilenamePrefix = path.Join(
+			dir,
+			string(file.GoPackageName),
+			base,
+		)
+		g = gen.NewGeneratedFile(
+			file.GeneratedFilenamePrefix+"_grpc.pb.go",
+			protogen.GoImportPath(path.Join(
+				string(file.GoImportPath),
+				string(file.GoPackageName),
+			)),
+		)
+	}
 	// Attach all comments associated with the syntax field.
 	genLeadingComments(g, file.Desc.SourceLocations().ByPath(protoreflect.SourcePath{fileDescriptorProtoSyntaxFieldNumber}))
 	g.P("// Code generated by protoc-gen-go-grpc. DO NOT EDIT.")
diff --git a/cmd/protoc-gen-go-grpc/main.go b/cmd/protoc-gen-go-grpc/main.go
index 340eaf3e..7f0fba56 100644
--- a/cmd/protoc-gen-go-grpc/main.go
+++ b/cmd/protoc-gen-go-grpc/main.go
@@ -44,6 +44,7 @@ import (
 const version = "1.3.0"
 
 var requireUnimplemented *bool
+var separatePackage *bool
 
 func main() {
 	showVersion := flag.Bool("version", false, "print the version and exit")
@@ -55,6 +56,7 @@ func main() {
 
 	var flags flag.FlagSet
 	requireUnimplemented = flags.Bool("require_unimplemented_servers", true, "set to false to match legacy behavior")
+	separatePackage = flags.Bool("separate_package", false, "set to true to write generated files to a separate grpc package")
 
 	protogen.Options{
 		ParamFunc: flags.Set,
EOF
WORKDIR /tmp/grpc-go/cmd/protoc-gen-go-grpc
RUN --mount=type=cache,target=/go/pkg/mod \
    go build -o protoc-gen-go-grpc -ldflags "-s -w" -trimpath

FROM scratch
COPY --from=build --link /etc/passwd /etc/passwd
COPY --from=build --link --chown=root:root /tmp/grpc-go/cmd/protoc-gen-go-grpc/protoc-gen-go-grpc .
USER nobody
ENTRYPOINT [ "/protoc-gen-go-grpc" ]
