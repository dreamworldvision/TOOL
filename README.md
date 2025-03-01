import { useDropzone } from 'react-dropzone';

const PDFUploader = ({ onUpload }) => {
  const { getRootProps, getInputProps } = useDropzone({
    accept: 'application/pdf',
    onDrop: (files) => onUpload(files),
  });

  return (
    <div {...getRootProps()} className="p-4 border-dashed border-2">
      <input {...getInputProps()} />
      <p>Drag PDFs here or click to upload</p>
    </div>
  );
};


