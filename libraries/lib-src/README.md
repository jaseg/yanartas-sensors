# Library sources

`kicad-footprint-generator_yanartas_custom.yaml` holds the package definitions for the
footprints in `libraries/HighSpeed_Interface/HighSpeed_Interface.pretty`,
`libraries/Gowin_FPGA_KiCad/Gowin_FPGA.pretty` and `libraries/Gowin_FPGA_KiCad/Power_Management.pretty`.
Every value is transcribed from the manufacturer drawing named in `size_source`.

To regenerate (footprints and STEP models) with the official KiCad generator
(https://gitlab.com/kicad/libraries/kicad-footprint-generator, tested at commit 090252c):

    cp kicad-footprint-generator_yanartas_custom.yaml <generator>/data/package/no_lead/yanartas_custom.yaml
    cd <generator>
    python src/generators/generate.py --generator package/no_lead \
        --part 'Diodes_X1-QFN-32*' 'Diodes_TQFN-52*' 'Texas_RKS0020A*' 'Gowin_QFN-48*' \
               'QFN-20-1EP_3x3mm_P0.4mm_EP1.7x1.7mm' \
        --output-dir-footprints out --output-dir-models out

The generated model paths point at `${KICAD10_3DMODEL_DIR}/yanartas_generated.3dshapes/`;
they were rewritten to `${KIPRJMOD}/../libraries/<lib dir>/<lib>.3dshapes/` so the board projects in
`power/` and `sensing/` find them.

TI RET0040A (TMUXHS4446) uses the stock KiCad footprint
`Package_DFN_QFN:Texas_WQFN-40-1EP_3x6mm_P0.4mm_EP1.7x4.5mm` instead.

Note: Diodes' XEA32 land pattern (PI3HDX12311) joins the middle pad of each short side
(pins 14 and 30, both GND) to the exposed pad with a narrow copper strip. The generated
footprint uses plain pads; draw that connection in the layout.
