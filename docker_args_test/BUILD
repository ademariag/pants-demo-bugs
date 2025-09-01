docker_image(
    name="base",
    source="Dockerfile.base",
)

docker_image(
    name="base2",
    source="Dockerfile.base2",
)

docker_image(
    name="main",
    source="Dockerfile",
    extra_build_args=[
        "BASE_IMAGE=:base"
    ]
)

docker_image(
    name="main-alt",
    source="Dockerfile",
    extra_build_args=[
        "BASE_IMAGE=:base2"
    ]
)