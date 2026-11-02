# Pick source file to flash
if (NOT DEFINED APP_VARIANT)
    set(APP_VARIANT "a")
endif ()

if (APP_VARIANT STREQUAL "a")
    set(MAIN_SRC "hello_world_main_variant_a.c")
elseif (APP_VARIANT STREQUAL "b")
    set(MAIN_SRC "hello_world_main_variant_b.c")
else ()
    message(FATAL_ERROR "APP_VARIANT must be either 'a' or 'b', got '${APP_VARIANT}'")
endif ()

idf_component_register(SRCS ${MAIN_SRC}
        PRIV_REQUIRES spi_flash app_update esp_partition
        INCLUDE_DIRS ""
)
