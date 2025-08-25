__defaults__(
    {
        "docker_image": dict(
            source="Dockerfile",
            build_platform=["linux/amd64"],
            image_tags=[
                "{pants.hash}",
                env("RANDOM_TAG")
            ],
        )
    }
)

docker_image(
    name="base",
    source="Dockerfile.base",
)

docker_image(
    name="main",
    source="Dockerfile",
)